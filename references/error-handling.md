# Axum Error Handling Reference (axum 0.8.x)

## 1. Result<T, E> in Handlers

Axum handlers return `impl IntoResponse`. Because `Result<T, E>` has a blanket `IntoResponse` impl when both `T` and `E` implement `IntoResponse`, handlers can return `Result<impl IntoResponse, impl IntoResponse>` and use `?` to propagate errors naturally.

```rust
use axum::{response::IntoResponse, http::StatusCode, Json};
use serde_json::{json, Value};

async fn handler() -> Result<Json<Value>, StatusCode> {
    let data = fetch_data().await?;
    Ok(Json(json!({ "result": data })))
}

async fn fetch_data() -> Result<String, StatusCode> {
    Err(StatusCode::INTERNAL_SERVER_ERROR) // ? propagates this up
}
```

**Why this works:** Axum provides `impl IntoResponse for Result<T, E>`. When the handler returns `Ok(value)`, axum calls `value.into_response()`. When it returns `Err(error)`, axum calls `error.into_response()`. This means any type that implements `IntoResponse` can be used on either side of a `Result`.

## 2. Custom Error Types

Implement `IntoResponse` on your own enums or structs to produce rich error responses. This lets you control both the HTTP status code and the response body.

```rust
use axum::{response::{IntoResponse, Response}, http::StatusCode, Json};
use serde_json::json;

enum AppError {
    NotFound(String),
    Unauthorized,
    Internal(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::NotFound(msg) => (StatusCode::NOT_FOUND, msg),
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, "unauthorized".into()),
            AppError::Internal(msg) => (StatusCode::INTERNAL_SERVER_ERROR, msg),
        };
        (status, Json(json!({ "error": message }))).into_response()
    }
}

// Now handlers return Result<Json<Value>, AppError>
async fn get_user() -> Result<Json<serde_json::Value>, AppError> {
    Err(AppError::NotFound("user not found".into()))
}
```

**Why implement IntoResponse directly?** It gives full control over the status code and body in one place. The alternative is wrapping every error in a tuple `(StatusCode, Json<…>)` at each call site, which is repetitive and error-prone.

## 3. Implementing Display and Error

When you implement `std::error::Error`, the trait requires `Display`. Both are needed because:

- **`Display`** — provides the human-readable message; used by ` anyhow`, `tracing`, and any code that formats the error.
- **`Error`** — marks the type as an error in the Rust ecosystem; enables `source()` chaining, `Box<dyn Error>`, and `anyhow` compatibility.

```rust
use std::fmt;

#[derive(Debug)]
enum AppError {
    NotFound(String),
    Database(sqlx::Error),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::NotFound(msg) => write!(f, "not found: {msg}"),
            AppError::Database(e) => write!(f, "database error: {e}"),
        }
    }
}

impl std::error::Error for AppError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            AppError::NotFound(_) => None,
            AppError::Database(e) => Some(e), // lets callers inspect the inner error
        }
    }
}
```

## 4. Rejection Types

Every extractor (except `Extension`) produces a _rejection_ type when it fails. These rejections implement `IntoResponse`, so unhandled extraction failures still produce a valid HTTP response—but the default body is usually a plain string.

```rust
use axum::{
    extract::{Path, Query, Json},
    http::StatusCode,
    response::IntoResponse,
};
use serde::Deserialize;

// PathRejection when path params don't parse
async fn user_path(Path(id): Path<u32>) -> String { format!("{id}") }

// QueryRejection when query string is missing or malformed
#[derive(Deserialize)]
struct Params { page: u32 }
async fn list(Query(p): Query<Params>) -> String { format!("{}", p.page) }

// JsonRejection when body is invalid JSON or wrong shape
#[derive(Deserialize)]
struct CreateUser { name: String }
async fn create(Json(user): Json<CreateUser>) -> String { user.name }

// Catch and customize rejections by returning Result:
async fn safe_create(Json(user): Json<CreateUser>) -> Result<String, (StatusCode, String)> {
    // If Json extraction fails, axum returns Err(JsonRejection as (StatusCode, String))
    Ok(user.name)
}
```

**Why rejections exist:** Axum extractors are fallible. Rather than panicking on bad input, each extractor returns a typed rejection that implements `IntoResponse`. This keeps the extraction logic decoupled from error formatting.

## 5. WithRejection (axum-extra)

`WithRejection` wraps any extractor to customize its rejection response. Use it to return structured JSON error bodies instead of plain text.

```rust
use axum_extra::extract::WithRejection;
use axum::extract:: rejection::JsonRejection;
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use serde_json::json;

// Define a custom rejection type
struct ApiRejection(Response);

impl From<JsonRejection> for ApiRejection {
    fn from(rejection: JsonRejection) -> Self {
        let (status, message) = match rejection {
            JsonRejection::JsonDataError(e) => (StatusCode::BAD_REQUEST, e.body_text()),
            JsonRejection::JsonSyntaxError(e) => (StatusCode::BAD_REQUEST, e.body_text()),
            JsonRejection::MissingJsonContentType(_) => {
                (StatusCode::UNSUPPORTED_MEDIA_TYPE, "expected Content-Type: application/json".into())
            }
            other => (StatusCode::INTERNAL_SERVER_ERROR, other.to_string()),
        };
        let body = json!({ "error": message });
        ApiRejection((status, axum::Json(body)).into_response())
    }
}

impl IntoResponse for ApiRejection {
    fn into_response(self) -> Response { self.0 }
}

// Use WithRejection to apply it
async fn create_user(WithRejection(Json(user), _): WithRejection<axum::Json<CreateUser>, ApiRejection>) -> String {
    user.name
}
```

**Why WithRejection?** Without it, customizing rejection responses requires wrapping every extractor in `Result` and matching on the rejection type inside each handler. `WithRejection` centralizes this logic once per extractor.

## 6. BoxError

`Box<dyn Error + Send + Sync>` (aliased as `BoxError` in axum) erases concrete error types. This is useful when a handler can fail with many unrelated error types and you don't need to match on them later.

```rust
use axum::{response::IntoResponse, http::StatusCode, BoxError};
use std::error::Error;

async fn polyglot_handler() -> Result<String, BoxError> {
    let db_result: Result<String, sqlx::Error> = Err(sqlx::Error::RowNotFound);
    let db_data = db_result?; // sqlx::Error -> BoxError via ? coercion

    let io_result: Result<Vec<u8>, std::io::Error> = Err(std::io::Error::new(
        std::io::ErrorKind::NotFound, "file missing"
    ));
    let _bytes = io_result?; // std::io::Error -> BoxError

    Ok(db_data)
}
```

**Why BoxError is a trade-off:** It avoids defining a large enum of error variants but sacrifices type information. Callers cannot `match` on the inner error. Reserve it for prototypes, fallback handlers, or cases where the error is only logged, never inspected.

## 7. Error Propagation Patterns

### The ? operator

The `?` operator calls `From::from` on the error, so define `impl From<LibError> for AppError` to let `?` perform conversions automatically.

```rust
impl From<sqlx::Error> for AppError {
    fn from(e: sqlx::Error) -> Self { AppError::Database(e) }
}

impl From<std::io::Error> for AppError {
    fn from(e: std::io::Error) -> Self { AppError::Internal(e.to_string()) }
}

// Now ? just works:
async fn read_config() -> Result<String, AppError> {
    let config = tokio::fs::read_to_string("config.toml").await?; // io::Error -> AppError
    Ok(config)
}
```

### map_err

Use `map_err` when a one-off conversion doesn't warrant a `From` impl.

```rust
async fn one_off() -> Result<String, AppError> {
    let data = some_lib::fetch()
        .await
        .map_err(|e| AppError::NotFound(format!("upstream: {e}")))?;
    Ok(data)
}
```

### anyhow integration

```rust
use anyhow::Result;

async fn anyhow_handler() -> Result<String> {
    let data = tokio::fs::read_to_string("config.toml").await?;
    Ok(data)
}

// anyhow::Error -> IntoResponse via a thin wrapper:
struct AppError(anyhow::Error);

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (StatusCode::INTERNAL_SERVER_ERROR, self.0.to_string()).into_response()
    }
}

impl From<anyhow::Error> for AppError {
    fn from(e: anyhow::Error) -> Self { AppError(e) }
}
```

## 8. Combining Multiple Error Types with thiserror

`thiserror` derives `Display`, `Error`, and `From` automatically, reducing boilerplate for error enums.

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum AppError {
    #[error("not found: {0}")]
    NotFound(String),

    #[error("unauthorized")]
    Unauthorized,

    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("validation: {0}")]
    Validation(String),

    #[error("internal error: {0}")]
    Internal(#[from] anyhow::Error),
}

// Each #[from] generates an impl From<T> for AppError automatically.
// The #[error("...")] attribute generates Display.

// Status mapping lives in IntoResponse:
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound(_) => (StatusCode::NOT_FOUND, self.to_string()),
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, self.to_string()),
            AppError::Validation(_) => (StatusCode::BAD_REQUEST, self.to_string()),
            AppError::Database(_) | AppError::Internal(_) => {
                (StatusCode::INTERNAL_SERVER_ERROR, self.to_string())
            }
        };
        (status, Json(json!({ "error": message }))).into_response()
    }
}
```

**Why thiserror?** Manual `Display` + `Error` + `From` impls for enums with many variants are verbose and repetitive. `thiserror` generates all three from a single declarative attribute, and the `#[from]` attribute provides automatic `From` conversions for the `?` operator.

## 9. Fallback Error Handlers

Register a fallback handler on the router to catch all unhandled errors and produce a consistent response.

```rust
use axum::{routing::get, Router, response::IntoResponse, http::StatusCode};
use serde_json::json;

async fn fallback() -> impl IntoResponse {
    (StatusCode::NOT_FOUND, Json(json!({ "error": "resource not found" })))
}

let app = Router::new()
    .route("/users", get(list_users))
    .fallback(fallback);
```

For errors from middleware or handlers that return `BoxError`:

```rust
async fn handle_box_error(err: BoxError) -> impl IntoResponse {
    if err.is::<tower::timeout::error::Elapsed>() {
        return (StatusCode::REQUEST_TIMEOUT, Json(json!({ "error": "request timed out" }))).into_response();
    }
    (
        StatusCode::INTERNAL_SERVER_ERROR,
        Json(json!({ "error": "internal server error" })),
    ).into_response()
}

// Attach via middleware:
// .layer(HandleErrorLayer::new(handle_box_error))
```

## 10. axum-extra ErrorResponse

`axum_extra::response::ErrorResponse` provides a structured error response type that implements `IntoResponse`.

```rust
use axum_extra::response::ErrorResponse;
use axum::http::StatusCode;
use axum::response::IntoResponse;

// ErrorResponse can be returned from IntoResponse directly:
fn bad_request() -> ErrorResponse {
    ErrorResponse::from(StatusCode::BAD_REQUEST)
}

// It pairs naturally with custom error types:
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let status = match &self {
            AppError::NotFound(_) => StatusCode::NOT_FOUND,
            AppError::Unauthorized => StatusCode::UNAUTHORIZED,
            AppError::Validation(_) => StatusCode::BAD_REQUEST,
            _ => StatusCode::INTERNAL_SERVER_ERROR,
        };
        ErrorResponse::from(status).into_response()
    }
}
```

## 11. Common Patterns

### API error enum with per-variant status codes

```rust
#[derive(Error, Debug)]
enum ApiError {
    #[error("{0}")]
    BadRequest(String),
    #[error("unauthorized")]
    Unauthorized,
    #[error("{0}")]
    NotFound(String),
    #[error("forbidden")]
    Forbidden,
    #[error("internal: {0}")]
    Internal(#[from] anyhow::Error),
}

impl ApiError {
    fn status(&self) -> StatusCode {
        match self {
            ApiError::BadRequest(_) => StatusCode::BAD_REQUEST,
            ApiError::Unauthorized => StatusCode::UNAUTHORIZED,
            ApiError::NotFound(_) => StatusCode::NOT_FOUND,
            ApiError::Forbidden => StatusCode::FORBIDDEN,
            ApiError::Internal(_) => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let status = self.status();
        (status, Json(json!({ "error": self.to_string() }))).into_response()
    }
}
```

### Database error to AppError conversion

```rust
impl From<sqlx::Error> for AppError {
    fn from(err: sqlx::Error) -> Self {
        match err {
            sqlx::Error::RowNotFound => AppError::NotFound("record not found".into()),
            sqlx::Error::Database(db_err) if db_err.code().as_deref() == Some("23505") => {
                AppError::Conflict("duplicate key".into())
            }
            other => AppError::Internal(other.to_string()),
        }
    }
}
```

### Validation errors with field-level detail

```rust
#[derive(Error, Debug)]
enum ValidationError {
    #[error("field '{field}': {message}")]
    Field { field: String, message: String },
    #[error("{0}")]
    Multiple(Vec<ValidationError>),
}

impl IntoResponse for ValidationError {
    fn into_response(self) -> Response {
        let body = match &self {
            ValidationError::Field { field, message } => {
                json!({ "errors": [{ "field": field, "message": message }] })
            }
            ValidationError::Multiple(errors) => {
                let items: Vec<_> = errors.iter().map(|e| json!({
                    "field": match e {
                        ValidationError::Field { field, .. } => field,
                        _ => "_general",
                    },
                    "message": e.to_string(),
                })).collect();
                json!({ "errors": items })
            }
        };
        (StatusCode::UNPROCESSABLE_ENTITY, Json(body)).into_response()
    }
}
```

### Error logging in middleware

```rust
use axum::{
    body::Body,
    http::{Request, StatusCode},
    middleware::Next,
    response::{IntoResponse, Response},
};

async fn error_logging_middleware(req: Request<Body>, next: Next) -> Response {
    let response = next.run(req).await;
    let status = response.status();
    if status.is_server_error() {
        tracing::error!(status = %status, "server error occurred");
    } else if status.is_client_error() {
        tracing::warn!(status = %status, "client error occurred");
    }
    response
}

// Apply: Router::new().layer(middleware::from_fn(error_logging_middleware))
```

## 12. HTTP Status Code Mapping

| Error Category     | Status Code | When to Use                                            |
| ------------------ | ----------- | ------------------------------------------------------ |
| Bad input syntax   | 400         | Malformed JSON, invalid query params                   |
| Missing auth       | 401         | No token / expired token                               |
| Forbidden          | 403         | Valid auth but insufficient permissions                |
| Not found          | 404         | Resource does not exist                                |
| Conflict           | 409         | Unique constraint violation, duplicate create          |
| Validation failure | 422         | Semantically invalid (e.g., email format, field rules) |
| Rate limited       | 429         | Too many requests                                      |
| Internal error     | 500         | Unexpected failure (never expose internals to clients) |
| Gateway timeout    | 504         | Upstream service did not respond in time               |

**Rule of thumb:** Client errors (4xx) mean the request is at fault; server errors (5xx) mean your code or infrastructure is at fault. Never return 500 for bad input, and never return 400 for a missing database record.

## Common Pitfalls

1. **Forgetting `impl IntoResponse` on the error type.** If `E` does not implement `IntoResponse`, `Result<T, E>` won't implement `IntoResponse` either, and the handler won't compile. The error message can be confusing—look for "the trait `IntoResponse` is not implemented for `Result<…, YourError>`".

2. **Implementing `Display` but not `Error`.** If you `#[derive(Error)]` via thiserror this is handled automatically. If you implement `IntoResponse` by hand on an enum that doesn't implement `std::error::Error`, you lose `source()` chaining and `anyhow` compatibility.

3. **Returning `Result<_, anyhow::Error>` directly.** `anyhow::Error` does not implement `IntoResponse`. Wrap it: define `AppError(anyhow::Error)` with a `From` impl and `IntoResponse` on the wrapper.

4. **Exposing internal errors to clients.** In the `IntoResponse` impl for `AppError::Internal`, log the full error with `tracing::error!` but return a generic message like "internal server error" in the response body. Leaking stack traces or SQL error messages to clients is a security risk.

5. **Not implementing `From` for extractor rejections.** If a handler uses `Path<T>` and `Json<U>` and returns `Result<_, AppError>`, you need `impl From<PathRejection> for AppError` and `impl From<JsonRejection> for AppError` (or use `WithRejection`) so that `?` works inside handlers that use extractors.

6. **Shadowing `std::error::Error`.** If you define a struct called `Error` in your crate root and then write `impl std::error::Error for Error {}`, name conflicts can cause confusing compile errors. Prefer names like `AppError` or `ApiError`.
