# Testing Axum 0.8.x Applications

## Cargo.toml dev-dependencies

Add these as `[dev-dependencies]` — they exist only when running `cargo test` so they never bloat production builds. `tower` provides `ServiceExt` for driving the router synchronously in tests. `http-body-util` lets you read response bodies as bytes. `serde_json` builds and asserts JSON payloads.

```toml
[dev-dependencies]
tower = { version = "0.5", features = ["util"] }
http-body-util = "0.1"
serde_json = "1"
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
reqwest = { version = "0.12", features = ["json"] }
```

## #[cfg(test)] Module Organization

Place tests inside the same file using a `#[cfg(test)]` module. This keeps tests next to the code they exercise and gives them access to private functions. For larger projects, split into a `tests/` directory for integration tests that import your crate as an external user would.

```rust
// src/handlers.rs
pub async fn greet(Path(name): Path<String>) -> impl IntoResponse {
    format!("Hello, {name}!")
}

#[cfg(test)]
mod tests {
    use super::*;
    use axum::{
        body::Body,
        extract::Path,
        http::{Request, StatusCode},
        routing::get,
        Router,
    };
    use tower::ServiceExt; // for oneshot()

    #[tokio::test]
    async fn test_greet() {
        let app = Router::new().route("/hello/{name}", get(greet));
        let res = app
            .oneshot(Request::builder().uri("/hello/world").body(Body::empty()).unwrap())
            .await
            .unwrap();
        assert_eq!(res.status(), StatusCode::OK);
    }
}
```

## Unit Testing with tower::ServiceExt

`tower::ServiceExt::oneshot()` sends a single request through the entire middleware + handler stack and returns the response. It works because `Router` implements `tower::Service<Request>`, meaning the router itself _is_ a service that can be polled for a response. `oneshot()` handles the async poll loop for you in one call.

```rust
use axum::{body::Body, routing::get, Router};
use http::Request;
use tower::ServiceExt;

async fn index() -> &'static str {
    "ok"
}

#[tokio::test]
async fn basic_oneshot() {
    let app = Router::new().route("/", get(index));

    let response = app
        .oneshot(
            Request::builder()
                .uri("/")
                .body(Body::empty())
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), 200);
}
```

## Test State Setup

Inject lightweight fakes or in-memory stand-ins for `AppState`. Axum passes state via `Arc`, so cloning is cheap. This lets every test get a fresh, isolated state instance with no shared mutable data between tests — eliminating flakiness.

```rust
use std::sync::Arc;

#[derive(Clone)]
struct AppState {
    db: Arc<std::collections::HashMap<String, String>>,
}

// Production handler uses the state:
async fn get_user(
    axum::extract::State(state): axum::extract::State<AppState>,
    axum::extract::Path(id): axum::extract::Path<String>,
) -> impl axum::response::IntoResponse {
    match state.db.get(&id) {
        Some(name) => format!("User: {name}"),
        None => axum::http::StatusCode::NOT_FOUND.into_response(),
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use axum::{body::Body, http::Request, routing::get, Router};
    use http_body_util::BodyExt;
    use std::collections::HashMap;
    use tower::ServiceExt;

    fn test_app() -> (Router, AppState) {
        let mut db = HashMap::new();
        db.insert("1".into(), "Alice".into());
        let state = AppState { db: Arc::new(db) };
        let app = Router::new()
            .route("/users/{id}", get(get_user))
            .with_state(state.clone());
        (app, state)
    }

    #[tokio::test]
    async fn returns_user() {
        let (app, _) = test_app();
        let res = app
            .oneshot(Request::builder().uri("/users/1").body(Body::empty()).unwrap())
            .await
            .unwrap();
        let body = res.into_body().collect().await.unwrap().to_bytes();
        assert_eq!(&body[..], b"User: Alice");
    }

    #[tokio::test]
    async fn missing_user_returns_404() {
        let (app, _) = test_app();
        let res = app
            .oneshot(Request::builder().uri("/users/999").body(Body::empty()).unwrap())
            .await
            .unwrap();
        assert_eq!(res.status(), 404);
    }
}
```

## Testing Request Extraction

Axum extractors reject requests that don't match their expectations, returning 400 or 422 automatically. Test each extractor boundary: missing fields, wrong types, and valid payloads. Use `serde_json::json!` to build request bodies concisely.

```rust
use axum::{extract::Query, Json};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct Pagination {
    page: u32,
    per_page: Option<u32>,
}

async fn list_items(Query(p): Query<Pagination>) -> String {
    let per = p.per_page.unwrap_or(10);
    format!("page={} per_page={}", p.page, per)
}

#[derive(Deserialize, Serialize)]
struct CreateUser {
    name: String,
    email: String,
}

async fn create_user(Json(input): Json<CreateUser>) -> (axum::http::StatusCode, Json<CreateUser>) {
    (axum::http::StatusCode::CREATED, Json(input))
}

#[cfg(test)]
mod extraction_tests {
    use super::*;
    use axum::{body::Body, http::Request, routing::{get, post}, Router};
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    #[tokio::test]
    async fn query_with_optional_field() {
        let app = Router::new().route("/items", get(list_items));
        let res = app
            .oneshot(
                Request::builder()
                    .uri("/items?page=2")
                    .body(Body::empty())
                    .unwrap(),
            )
            .await
            .unwrap();
        let body = res.into_body().collect().await.unwrap().to_bytes();
        assert_eq!(&body[..], b"page=2 per_page=10");
    }

    #[tokio::test]
    async fn json_extraction_valid() {
        let app = Router::new().route("/users", post(create_user));
        let body = serde_json::json!({"name": "Bob", "email": "bob@example.com"}).to_string();
        let res = app
            .oneshot(
                Request::builder()
                    .method("POST")
                    .uri("/users")
                    .header("content-type", "application/json")
                    .body(Body::from(body))
                    .unwrap(),
            )
            .await
            .unwrap();
        assert_eq!(res.status(), 201);
    }

    #[tokio::test]
    async fn json_extraction_missing_field_returns_422() {
        let app = Router::new().route("/users", post(create_user));
        // Missing required "email" field
        let body = serde_json::json!({"name": "Bob"}).to_string();
        let res = app
            .oneshot(
                Request::builder()
                    .method("POST")
                    .uri("/users")
                    .header("content-type", "application/json")
                    .body(Body::from(body))
                    .unwrap(),
            )
            .await
            .unwrap();
        assert_eq!(res.status(), 422);
    }
}
```

## Testing Response Types

Deserialize response bodies into typed structs to assert both structure and values. This catches schema regressions where the handler returns valid JSON but the wrong shape.

```rust
use axum::Json;
use serde::Serialize;

#[derive(Serialize)]
struct HealthResponse {
    status: String,
    version: String,
}

async fn health() -> Json<HealthResponse> {
    Json(HealthResponse {
        status: "ok".into(),
        version: env!("CARGO_PKG_VERSION").into(),
    })
}

#[cfg(test)]
mod response_tests {
    use super::*;
    use axum::{body::Body, http::Request, routing::get, Router};
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    #[tokio::test]
    async fn health_returns_expected_json_structure() {
        let app = Router::new().route("/health", get(health));
        let res = app
            .oneshot(Request::builder().uri("/health").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(res.status(), 200);
        assert_eq!(
            res.headers().get("content-type").unwrap(),
            "application/json"
        );

        let bytes = res.into_body().collect().await.unwrap().to_bytes();
        let body: HealthResponse = serde_json::from_slice(&bytes).unwrap();
        assert_eq!(body.status, "ok");
        assert!(!body.version.is_empty());
    }
}
```

## Testing Error Cases

Test that handlers return correct status codes and error payloads when given bad input or when internal failures occur. Custom error types that implement `IntoResponse` should be tested to ensure the serialized error body matches your API contract.

```rust
use axum::response::{IntoResponse, Response};
use serde::Serialize;

#[derive(Serialize)]
struct ApiError {
    error: String,
    code: u16,
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let status = axum::http::StatusCode::from_u16(self.code).unwrap_or(axum::http::StatusCode::INTERNAL_SERVER_ERROR);
        (status, Json(self)).into_response()
    }
}

async fn divide(Path((a, b)): Path<(i32, i32)>) -> Result<String, ApiError> {
    if b == 0 {
        return Err(ApiError { error: "division by zero".into(), code: 400 });
    }
    Ok(format!("{}", a / b))
}

#[cfg(test)]
mod error_tests {
    use super::*;
    use axum::{body::Body, extract::Path, http::Request, routing::get, Router};
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    #[tokio::test]
    async fn division_by_zero_returns_400_with_error_body() {
        let app = Router::new().route("/divide/{a}/{b}", get(divide));
        let res = app
            .oneshot(Request::builder().uri("/divide/10/0").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(res.status(), 400);
        let bytes = res.into_body().collect().await.unwrap().to_bytes();
        let err: ApiError = serde_json::from_slice(&bytes).unwrap();
        assert_eq!(err.error, "division by zero");
        assert_eq!(err.code, 400);
    }

    #[tokio::test]
    async fn unknown_route_returns_404() {
        let app = Router::new().route("/divide/{a}/{b}", get(divide));
        let res = app
            .oneshot(Request::builder().uri("/nonexistent").body(Body::empty()).unwrap())
            .await
            .unwrap();
        assert_eq!(res.status(), 404);
    }
}
```

## Integration Testing Patterns

Test the full request lifecycle across multiple handlers by building a complete `Router` with shared state. Verify that creating a resource in one call makes it available in a subsequent call — this catches issues with state consistency, middleware side effects, and routing.

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

#[derive(Clone)]
struct AppState {
    items: Arc<RwLock<Vec<String>>>,
}

async fn create_item(
    axum::extract::State(state): axum::extract::State<AppState>,
    axum::extract::Path(name): axum::extract::Path<String>,
) -> impl axum::response::IntoResponse {
    state.items.write().await.push(name.clone());
    (axum::http::StatusCode::CREATED, format!("Created: {name}"))
}

async fn list_items(
    axum::extract::State(state): axum::extract::State<AppState>,
) -> impl axum::response::IntoResponse {
    let items = state.items.read().await;
    axum::Json(items.clone())
}

#[cfg(test)]
mod integration_tests {
    use super::*;
    use axum::{body::Body, http::Request, routing::{get, post}, Router};
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    fn app() -> Router {
        let state = AppState { items: Arc::new(RwLock::new(Vec::new())) };
        Router::new()
            .route("/items", post(create_item))
            .route("/items", get(list_items))
            .with_state(state)
    }

    #[tokio::test]
    async fn create_then_list() {
        let app = app();

        // Create two items
        let res = app
            .clone()
            .oneshot(
                Request::builder()
                    .method("POST")
                    .uri("/items/apple")
                    .body(Body::empty())
                    .unwrap(),
            )
            .await
            .unwrap();
        assert_eq!(res.status(), 201);

        let res = app
            .clone()
            .oneshot(
                Request::builder()
                    .method("POST")
                    .uri("/items/banana")
                    .body(Body::empty())
                    .unwrap(),
            )
            .await
            .unwrap();
        assert_eq!(res.status(), 201);

        // List all items
        let res = app
            .oneshot(
                Request::builder()
                    .method("GET")
                    .uri("/items")
                    .body(Body::empty())
                    .unwrap(),
            )
            .await
            .unwrap();
        assert_eq!(res.status(), 200);

        let bytes = res.into_body().collect().await.unwrap().to_bytes();
        let items: Vec<String> = serde_json::from_slice(&bytes).unwrap();
        assert_eq!(items, vec!["apple", "banana"]);
    }
}
```

## Testing Middleware

Test middleware in isolation by asserting headers, status codes, or body changes that the middleware introduces. Auth middleware should reject unauthenticated requests; logging middleware should add tracking headers. Build the app with `.layer()` and verify the side effects.

```rust
use axum::{
    extract::Request as AxumRequest,
    middleware::{self, Next},
    response::Response,
};

// Middleware that adds a custom header to every response
async fn add_request_id(mut req: AxumRequest, next: Next) -> Response {
    let id = "test-request-123";
    req.headers_mut().insert("x-request-id", id.parse().unwrap());
    next.run(req).await
}

// Middleware that rejects requests without an Authorization header
async fn require_auth(
    req: AxumRequest,
    next: Next,
) -> Response {
    if req.headers().get("authorization").is_none() {
        return (axum::http::StatusCode::UNAUTHORIZED, "missing auth").into_response();
    }
    next.run(req).await
}

async fn protected() -> &'static str {
    "secret data"
}

#[cfg(test)]
mod middleware_tests {
    use super::*;
    use axum::{body::Body, http::Request, routing::get, Router};
    use tower::ServiceExt;

    #[tokio::test]
    async fn request_id_middleware_adds_header_to_request() {
        let app = Router::new()
            .route("/", get(|| async { "ok" }))
            .layer(middleware::from_fn(add_request_id));

        let res = app
            .oneshot(Request::builder().uri("/").body(Body::empty()).unwrap())
            .await
            .unwrap();
        assert_eq!(res.status(), 200);
    }

    #[tokio::test]
    async fn auth_middleware_rejects_missing_token() {
        let app = Router::new()
            .route("/protected", get(protected))
            .layer(middleware::from_fn(require_auth));

        let res = app
            .oneshot(Request::builder().uri("/protected").body(Body::empty()).unwrap())
            .await
            .unwrap();
        assert_eq!(res.status(), 401);
    }

    #[tokio::test]
    async fn auth_middleware_passes_with_valid_token() {
        let app = Router::new()
            .route("/protected", get(protected))
            .layer(middleware::from_fn(require_auth));

        let res = app
            .oneshot(
                Request::builder()
                    .uri("/protected")
                    .header("authorization", "Bearer token123")
                    .body(Body::empty())
                    .unwrap(),
            )
            .await
            .unwrap();
        assert_eq!(res.status(), 200);
    }
}
```

## Testing WebSocket

WebSocket upgrades cannot be fully tested with `oneshot()` because the connection upgrades to a raw TCP stream — the HTTP response is a 101 Switching Protocols handshake, not a final body. Test the _upgrade rejection_ path (e.g., missing headers) with `oneshot()`. For the live protocol, use `tokio-tungstenite` to connect to a real spawned server.

```rust
use axum::{
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    response::IntoResponse,
};

async fn ws_handler(ws: WebSocketUpgrade) -> impl IntoResponse {
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(Ok(msg)) = socket.recv().await {
        if msg.is_text() {
            let reply = format!("echo: {}", msg);
            let _ = socket.send(Message::Text(reply)).await;
        }
    }
}

#[cfg(test)]
mod ws_tests {
    use super::*;
    use axum::{body::Body, http::Request, routing::get, Router};
    use tower::ServiceExt;

    // Test that missing Upgrade header causes rejection
    #[tokio::test]
    async fn ws_rejects_normal_http_request() {
        let app = Router::new().route("/ws", get(ws_handler));
        let res = app
            .oneshot(Request::builder().uri("/ws").body(Body::empty()).unwrap())
            .await
            .unwrap();
        // Without proper WebSocket upgrade headers, the extractor fails
        assert_ne!(res.status(), 200);
    }
}
```

## Common Test Helpers

Extract repetitive setup into shared helper functions. Centralize app construction and request building so tests stay focused on assertions, not boilerplate.

```rust
#[cfg(test)]
pub(crate) mod test_helpers {
    use axum::{body::Body, routing::MethodRouter, Router};
    use http::{Request, StatusCode};
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    /// Build a request with a JSON body and correct Content-Type header.
    pub fn json_request<T: serde::Serialize>(
        method: &str,
        uri: &str,
        body: &T,
    ) -> Request<Body> {
        Request::builder()
            .method(method)
            .uri(uri)
            .header("content-type", "application/json")
            .body(Body::from(serde_json::to_string(body).unwrap()))
            .unwrap()
    }

    /// Build a request with no body (for GET, DELETE, etc.).
    pub fn empty_request(method: &str, uri: &str) -> Request<Body> {
        Request::builder()
            .method(method)
            .uri(uri)
            .body(Body::empty())
            .unwrap()
    }

    /// Send a request through a Router and return (status, body bytes).
    pub async fn send(app: Router, req: Request<Body>) -> (StatusCode, bytes::Bytes) {
        let res = app.oneshot(req).await.unwrap();
        let status = res.status();
        let body = res.into_body().collect().await.unwrap().to_bytes();
        (status, body)
    }

    /// Assert a response body equals a UTF-8 string.
    pub fn assert_body_text(bytes: &bytes::Bytes, expected: &str) {
        assert_eq!(&bytes[..], expected.as_bytes());
    }
}

// Usage in tests:
#[cfg(test)]
mod helper_usage {
    use super::test_helpers::*;
    use axum::{routing::get, Router, Json};
    use serde::{Deserialize, Serialize};

    async fn ping() -> &'static str { "pong" }

    #[derive(Serialize, Deserialize)]
    struct Echo { msg: String }

    async fn echo(Json(body): Json<Echo>) -> Json<Echo> { Json(body) }

    #[tokio::test]
    async fn ping_test() {
        let app = Router::new().route("/ping", get(ping));
        let (status, body) = send(app, empty_request("GET", "/ping")).await;
        assert_eq!(status, StatusCode::OK);
        assert_body_text(&body, "pong");
    }

    #[tokio::test]
    async fn echo_test() {
        let app = Router::new().route("/echo", axum::routing::post(echo));
        let req = json_request("POST", "/echo", &Echo { msg: "hello".into() });
        let (status, body) = send(app, req).await;
        assert_eq!(status, StatusCode::OK);
        let echo: Echo = serde_json::from_slice(&body).unwrap();
        assert_eq!(echo.msg, "hello");
    }
}
```
