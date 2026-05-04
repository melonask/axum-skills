# Axum 0.8.x Middleware Reference

Complete guide to writing, composing, and applying middleware in the axum 0.8.x Rust web framework.
All code examples are fully self-contained and runnable.

## Table of Contents

- [Axum 0.8.x Middleware Reference](#axum-08x-middleware-reference)
  - [Table of Contents](#table-of-contents)
  - [Why Axum Middleware Works the Way It Does](#why-axum-middleware-works-the-way-it-does)
  - [middleware::from_fn](#middlewarefrom_fn)
  - [middleware::from_fn_with_state](#middlewarefrom_fn_with_state)
  - [middleware::map_request](#middlewaremap_request)
  - [middleware::map_response](#middlewaremap_response)
  - [Applying Middleware to Specific Routes](#applying-middleware-to-specific-routes)
  - [Layer on Router vs MethodRouter](#layer-on-router-vs-methodrouter)
  - [Composing Multiple Middleware](#composing-multiple-middleware)
  - [Tower Layers as Middleware](#tower-layers-as-middleware)
  - [Custom Middleware with Extractors](#custom-middleware-with-extractors)
    - [Dynamic Configuration in Middleware (x402 integration)](#dynamic-configuration-in-middleware-x402-integration)
  - [Request/Response Modification Patterns](#requestresponse-modification-patterns)
    - [Adding Request IDs](#adding-request-ids)
    - [Logging Middleware](#logging-middleware)
  - [Error Handling in Middleware](#error-handling-in-middleware)
  - [Real-World Examples](#real-world-examples)
    - [Authentication / Authorization Middleware](#authentication--authorization-middleware)
    - [Rate Limiting Middleware](#rate-limiting-middleware)
    - [Request Timing Middleware](#request-timing-middleware)
    - [Body Size Check Middleware](#body-size-check-middleware)
  - [Common Pitfalls](#common-pitfalls)
    - [1. Middleware Ordering is Inverted with `.layer()`](#1-middleware-ordering-is-inverted-with-layer)
    - [2. Extracting the Request Body in Middleware](#2-extracting-the-request-body-in-middleware)
    - [3. State Mismatch with `from_fn_with_state`](#3-state-mismatch-with-from_fn_with_state)
    - [4. Double-Processing with `.route_layer()` and `.layer()`](#4-double-processing-with-route_layer-and-layer)
    - [5. Forgetting `IntoResponse` for Error Types](#5-forgetting-intoresponse-for-error-types)
    - [6. Async Block Captures in from_fn Closures](#6-async-block-captures-in-from_fn-closures)

---

## Why Axum Middleware Works the Way It Does

Axum builds on **tower**'s `Service` trait. A middleware in tower is a service that wraps another service:
it receives a request, can inspect/modify it, optionally calls the inner service (the handler), and then
can inspect/modify the response. Axum provides ergonomic helper functions that let you write plain
`async fn` middleware without implementing `Service` manually — the framework converts your function
into a tower `Layer` under the hood.

Key mental model: **middleware wraps the handler like layers of an onion**. The outermost layer runs
first on the way in (request) and last on the way out (response).

---

## middleware::from_fn

Use `middleware::from_fn` for middleware that does not need access to shared application state.
The function receives anything that implements `FromRequest` as arguments, plus a
`Next<RequestBody>` as the last argument.

```rust
use axum::{
    middleware,
    Router,
    http::{HeaderMap, HeaderValue, StatusCode},
    response::IntoResponse,
};

// Basic middleware: inspect a header and short-circuit if missing.
async fn require_api_key(
    req: axum::http::Request<axum::body::Body>,
    next: middleware::Next,
) -> impl IntoResponse {
    if req.headers().get("x-api-key").is_none() {
        return (StatusCode::UNAUTHORIZED, "missing x-api-key header").into_response();
    }
    next.run(req).await
}

// Basic middleware: add a response header to every successful request.
async fn add_server_header(
    mut req: axum::http::Request<axum::body::Body>,
    next: middleware::Next,
) -> impl IntoResponse {
    let mut response = next.run(req).await;
    response.headers_mut().insert(
        "x-server",
        HeaderValue::from_static("axum-app"),
    );
    response
}

let app = Router::new()
    .route("/protected", axum::routing::get(|| async { "secret data" }))
    .route_layer(middleware::from_fn(require_api_key))
    .route("/public", axum::routing::get(|| async { "hello" }))
    .layer(middleware::from_fn(add_server_header));
```

**Why `next.run(req).await`**: The `Next` value captures the remaining service chain. Calling `.run()`
passes the request body downstream to the actual handler. If you **don't** call `.run()`, the handler
never executes — this is how you short-circuit (e.g., return 401 early).

**Important**: Extractors consume parts of the request. If you extract `HeaderMap`, the headers are
consumed by your middleware. The request body passed to `.run()` must match what the handler expects.
Use `Request<Body>` or individual extractors carefully.

---

## middleware::from_fn_with_state

Use `from_fn_with_state` when your middleware needs read-only access to shared application state.
The state is cloned (so it must be `Clone`) and passed as the first argument.

```rust
use axum::{middleware, Router, extract::State};
use std::sync::Arc;

#[derive(Clone)]
struct AppState {
    allowed_api_keys: Arc<Vec<String>>,
}

// Middleware that checks the API key against a whitelist stored in state.
async fn check_api_key(
    State(state): State<AppState>,
    req: axum::http::Request<axum::body::Body>,
    next: middleware::Next,
) -> impl axum::response::IntoResponse {
    let api_key = req
        .headers()
        .get("x-api-key")
        .and_then(|v| v.to_str().ok());

    match api_key {
        Some(key) if state.allowed_api_keys.contains(&key.to_string()) => {
            next.run(req).await
        }
        _ => (
            axum::http::StatusCode::UNAUTHORIZED,
            "invalid or missing api key",
        ).into_response(),
    }
}

let state = AppState {
    allowed_api_keys: Arc::new(vec!["secret123".into(), "admin-key".into()]),
};

let app = Router::new()
    .route("/data", axum::routing::get(|| async { "protected data" }))
    .layer(middleware::from_fn_with_state(state.clone(), check_api_key))
    .with_state(state);
```

**Why `with_state` exists**: Unlike handlers which receive state through axum's extractor system
automatically, middleware layers are constructed _before_ the router is finalized and the state type
is known. The `with_state` variant lets you inject state at layer-construction time.

---

## middleware::map_request

Use `map_request` for pure request transformations — no branching, no short-circuiting.
It takes a closure/function that receives a `Request<Body>` and returns a `Request<Body>`.

```rust
use axum::{middleware, Router, routing::get};
use axum::http::{Request, header, HeaderValue};

// Add a custom header to every incoming request.
async fn add_tenant_header(mut req: Request<axum::body::Body>) -> Request<axum::body::Body> {
    req.headers_mut().insert(
        header::X_REQUEST_ID,
        HeaderValue::from_static("auto-generated-id"),
    );
    req
}

// Rewrite the URI path (useful for API versioning migrations).
async fn strip_api_prefix(mut req: Request<axum::body::Body>) -> Request<axum::body::Body> {
    let path = req.uri().path().to_string();
    if let Some(stripped) = path.strip_prefix("/api/v1") {
        let new_uri = format!("{}{}", stripped, req.uri().query().map_or("".to_string(), |q| format!("?{}", q)));
        *req.uri_mut() = new_uri.parse().unwrap();
    }
    req
}

let app = Router::new()
    .route("/users", get(|| async { "users list" }))
    .layer(middleware::map_request(add_tenant_header))
    .layer(middleware::map_request(strip_api_prefix));
```

**Why map_request instead of from_fn**: `map_request` cannot short-circuit or modify the response.
Use it when you only need to transform the request — it's simpler and communicates intent clearly.
The compiler enforces that the function must return a request (not a response), making accidental
short-circuits impossible.

---

## middleware::map_response

Use `map_response` for pure response transformations — adding headers, modifying status codes, etc.
It receives a `Response<Body>` and returns a `Response<Body>`.

```rust
use axum::{middleware, Router, routing::get, http::{StatusCode, header, HeaderValue}};

// Normalize all 4xx responses to a consistent JSON error format.
async fn normalize_error_response(mut res: axum::response::Response) -> axum::response::Response {
    if res.status().is_client_error() {
        res.headers_mut().insert(
            header::CONTENT_TYPE,
            HeaderValue::from_static("application/json"),
        );
    }
    res
}

// Add CORS headers to every response.
async fn add_cors_headers(mut res: axum::response::Response) -> axum::response::Response {
    let headers = res.headers_mut();
    headers.insert(header::ACCESS_CONTROL_ALLOW_ORIGIN, HeaderValue::from_static("*"));
    headers.insert(header::ACCESS_CONTROL_ALLOW_METHODS, HeaderValue::from_static("GET, POST, PUT, DELETE"));
    headers.insert(header::ACCESS_CONTROL_ALLOW_HEADERS, HeaderValue::from_static("Content-Type, Authorization"));
    res
}

let app = Router::new()
    .route("/items", get(|| async { "items" }))
    .layer(middleware::map_response(add_cors_headers))
    .layer(middleware::map_response(normalize_error_response));
```

**Why map_response runs in reverse order**: Tower layers wrap like an onion. The last `.layer()` call
is the outermost layer, so its `map_response` closure runs **last** on the way out. In the example
above, `add_cors_headers` (applied second) runs after `normalize_error_response` (applied first) on
response — meaning CORS headers are added to the already-normalized response.

---

## Applying Middleware to Specific Routes

Use `.route_layer()` to apply middleware only to routes defined on that specific `Router` segment.
Use `.layer()` to apply middleware to the entire router subtree. Scope with `.nest()` or chained
`Router::new()` calls to create different middleware zones.

```rust
use axum::{middleware, Router, routing::{get, post}, http::StatusCode};
use axum::extract::Extension;

async fn admin_auth(next: middleware::Next) -> impl axum::response::IntoResponse {
    next.run(()).await
}

async fn logging(next: middleware::Next) -> impl axum::response::IntoResponse {
    next.run(()).await
}

let app = Router::new()
    // These routes get ONLY the logging middleware (from the parent Router::merge below)
    .route("/health", get(|| async { "ok" }))
    .route("/version", get(|| async { "1.0" }))
    // This nested router gets BOTH admin_auth and logging
    .nest("/admin", {
        Router::new()
            .route("/dashboard", get(|| async { "admin dashboard" }))
            .route("/users", get(|| async { "admin users" }))
            .route_layer(middleware::from_fn(admin_auth))
    })
    // The logging layer wraps EVERYTHING above
    .layer(middleware::from_fn(logging));
```

**Why `.route_layer()` vs `.layer()`**: `.route_layer()` only applies to routes registered _before_ it
on that same `Router` builder — it does NOT affect routes added after it or nested routers added with
`.nest()` after it. `.layer()` wraps the entire router as a service, affecting everything.

---

## Layer on Router vs MethodRouter

`.layer()` on `Router` wraps the entire router. `.layer()` on a single `MethodRouter` (returned by
`get()`, `post()`, etc.) wraps only that one route handler.

```rust
use axum::{middleware, Router, routing::{get, post}};

async fn per_route_mw(next: middleware::Next) -> impl axum::response::IntoResponse {
    next.run(()).await
}

async fn global_mw(next: middleware::Next) -> impl axum::response::IntoResponse {
    next.run(()).await
}

let app = Router::new()
    // This layer applies ONLY to the /special route
    .route(
        "/special",
        get(|| async { "special" }).layer(middleware::from_fn(per_route_mw)),
    )
    // These routes get only the global middleware
    .route("/users", get(|| async { "users" }))
    .route("/items", post(|| async { "created" }))
    // This layer wraps EVERYTHING
    .layer(middleware::from_fn(global_mw));
```

**Why this matters**: Applying middleware per-route avoids unnecessary work for routes that don't
need it. An auth middleware, for example, should only run on protected routes, not on public health
checks. Order: per-route middleware runs _inside_ the global middleware layers.

---

## Composing Multiple Middleware

Use `tower::ServiceBuilder` to compose layers in a readable, explicit order. The builder applies
layers in declaration order: the first `.layer()` is the outermost.

```rust
use axum::{middleware, Router, routing::get};
use tower::ServiceBuilder;

async fn auth_mw(next: middleware::Next) -> impl axum::response::IntoResponse { next.run(()).await }
async fn log_mw(next: middleware::Next) -> impl axum::response::IntoResponse { next.run(()).await }
async fn compress_mw(next: middleware::Next) -> impl axum::response::IntoResponse { next.run(()).await }

let app = Router::new()
    .route("/", get(|| async { "hello" }))
    .layer(
        ServiceBuilder::new()
            // Runs FIRST on request, LAST on response
            .layer(middleware::from_fn(log_mw))
            // Runs SECOND on request, SECOND-TO-LAST on response
            .layer(middleware::from_fn(auth_mw))
            // Runs LAST on request, FIRST on response
            .layer(middleware::from_fn(compress_mw))
    );
```

**Why ServiceBuilder**: Without it, chaining `.layer().layer().layer()` is confusing because
subsequent layers wrap previous ones (the last `.layer()` is outermost). `ServiceBuilder` applies
in declaration order for clarity: first declared = outermost layer.

---

## Tower Layers as Middleware

Any type implementing `tower::Layer` works with `.layer()`. This includes all middleware from the
tower and tower-http ecosystems.

```rust
use axum::{Router, routing::get};
use tower_http::{
    compression::CompressionLayer,
    cors::CorsLayer,
    trace::TraceLayer,
    limit::RequestBodyLimitLayer,
    timeout::TimeoutLayer,
};
use std::time::Duration;
use tower::ServiceBuilder;

let app = Router::new()
    .route("/api", get(|| async { "response" }))
    .layer(
        ServiceBuilder::new()
            // Trace every request with tracing spans
            .layer(TraceLayer::new_for_http())
            // Enforce a 30-second timeout on every request
            .layer(TimeoutLayer::new(Duration::from_secs(30)))
            // Limit request body to 1 MB
            .layer(RequestBodyLimitLayer::new(1024 * 1024))
            // Gzip/br deflate compression
            .layer(CompressionLayer::new())
            // CORS configuration
            .layer(CorsLayer::permissive())
    );
```

**Why this is powerful**: The tower ecosystem provides production-ready middleware (rate limiting,
concurrency limits, load shedding, retries for outbound requests). Because axum is tower-native,
these all compose seamlessly without adapter code.

---

## Custom Middleware with Extractors

Middleware functions can use any axum extractor — `State`, `Extension`, `Json`, `Path`, etc.
Extractors are resolved from the request just like in handlers.

```rust
use axum::{
    middleware, Router, routing::get,
    extract::{State, Extension, Path},
    http::StatusCode,
};
use std::sync::Arc;

#[derive(Clone)]
struct AppState { db_pool: Arc<String> }

#[derive(Clone)]
struct User { id: u64, name: String }

// Middleware that extracts a user from a path param and attaches it via Extension.
async fn load_user(
    State(_state): State<AppState>,
    Path(user_id): Path<u64>,
    next: middleware::Next,
) -> impl axum::response::IntoResponse {
    // Simulate DB lookup
    let user = User {
        id: user_id,
        name: format!("user-{}", user_id),
    };
    // Insert the user into the request extensions so handlers can extract it
    let mut req = next.into_request();
    req.extensions_mut().insert(user);
    next.run(req).await
}

async fn handler(Extension(user): Extension<User>) -> String {
    format!("Hello, {} (id: {})", user.name, user.id)
}

let state = AppState { db_pool: Arc::new("postgres://...".into()) };

let app = Router::new()
    .route("/users/{id}/profile", get(handler))
    .route_layer(middleware::from_fn(load_user))
    .with_state(state);
```

**Why Extension is the bridge**: Extensions are a type map attached to the request. Middleware can
insert data into extensions, and downstream handlers can extract it with `Extension<T>`. This is the
standard pattern for passing middleware-computed data to handlers.

### Dynamic Configuration in Middleware (x402 integration)

When using complex middleware (like `x402-axum`) that needs to read dynamic configuration, use `axum::Extension` to inject the `Arc<Config>` so the middleware callback can access it:

```rust
let catalog: Arc<Catalog> = Arc::new(load_toml_catalog());

let app = Router::new()
    .route("/api/tasks/{kind}", get(handler))
    .layer(axum::Extension(catalog)) // Inject first!
    .layer(middleware::from_fn(|Extension(cat): Extension<Arc<Catalog>>, req, next| async move {
        // Use `cat` to dynamically set x402 prices
        next.run(req).await
    }));
```

---

## Request/Response Modification Patterns

### Adding Request IDs

```rust
use axum::{middleware, Router, routing::get};
use axum::http::{Request, header, HeaderValue};
use axum::extract::Extension;
use uuid::Uuid;

#[derive(Clone)]
struct RequestId(String);

async fn attach_request_id(mut req: Request<axum::body::Body>) -> Request<axum::body::Body> {
    let id = Uuid::new_v4().to_string();
    req.headers_mut().insert(
        header::X_REQUEST_ID,
        HeaderValue::from_str(&id).unwrap(),
    );
    req.extensions_mut().insert(RequestId(id));
    req
}

async fn handler(Extension(req_id): Extension<RequestId>) -> String {
    format!("your request id: {}", req_id.0)
}

let app = Router::new()
    .route("/", get(handler))
    .layer(middleware::map_request(attach_request_id));
```

### Logging Middleware

```rust
use axum::{middleware, Router, routing::get};
use std::time::Instant;

async fn logging_middleware(
    req: axum::http::Request<axum::body::Body>,
    next: middleware::Next,
) -> impl axum::response::IntoResponse {
    let method = req.method().clone();
    let uri = req.uri().clone();
    let start = Instant::now();

    let response = next.run(req).await;

    let elapsed = start.elapsed();
    let status = response.status();
    tracing::info!(
        method = %method, uri = %uri, status = status.as_u16(),
        elapsed_ms = elapsed.as_millis() as u64,
        "request completed"
    );

    response
}

let app = Router::new()
    .route("/", get(|| async { "hello" }))
    .layer(middleware::from_fn(logging_middleware));
```

---

## Error Handling in Middleware

Middleware that returns `impl IntoResponse` can return error types directly. If a middleware
function returns a `Result`, use `.into_response()` on both `Ok` and `Err` variants.

```rust
use axum::{middleware, Router, routing::get, http::StatusCode, response::IntoResponse};

#[derive(Debug)]
enum AuthError {
    MissingToken,
    InvalidToken,
}

impl IntoResponse for AuthError {
    fn into_response(self) -> axum::response::Response {
        let (status, message) = match self {
            AuthError::MissingToken => (StatusCode::UNAUTHORIZED, "missing auth token"),
            AuthError::InvalidToken => (StatusCode::FORBIDDEN, "invalid auth token"),
        };
        (status, message).into_response()
    }
}

async fn auth_middleware(
    headers: axum::http::HeaderMap,
    next: middleware::Next,
) -> Result<impl IntoResponse, AuthError> {
    let token = headers
        .get("authorization")
        .and_then(|v| v.to_str().ok())
        .ok_or(AuthError::MissingToken)?;

    if !token.starts_with("Bearer valid-") {
        return Err(AuthError::InvalidToken);
    }

    Ok(next.run(req).await)
}

// IMPORTANT: Use .route_layer() so the middleware error short-circuits without
// falling through to routes that don't match.
let app = Router::new()
    .route("/protected", get(|| async { "secret" }))
    .route_layer(middleware::from_fn(auth_middleware));
```

**Why errors propagate correctly**: When middleware returns `Err`, it is converted into a response
via `IntoResponse`. The handler is never called. This is the recommended pattern — avoid panics in
middleware, return proper error responses instead.

---

## Real-World Examples

### Authentication / Authorization Middleware

```rust
use axum::{
    extract::State,
    http::{HeaderMap, StatusCode},
    middleware, response::IntoResponse, routing::get, Router,
};
use std::sync::Arc;

#[derive(Clone)]
struct AuthState {
    jwt_secret: Arc<String>,
}

#[derive(Clone)]
struct AuthenticatedUser { user_id: u64, roles: Vec<String> }

async fn jwt_auth(
    State(state): State<AuthState>,
    req: axum::http::Request<axum::body::Body>,
    next: middleware::Next,
) -> impl IntoResponse {
    let token = match req.headers().get("authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
    {
        Some(t) => t,
        None => return (StatusCode::UNAUTHORIZED, "missing token").into_response(),
    };

    // Simulate JWT validation — in production use jsonwebtoken crate
    if token != "valid-token" {
        return (StatusCode::UNAUTHORIZED, "invalid token").into_response();
    }

    let user = AuthenticatedUser {
        user_id: 42,
        roles: vec!["admin".into(), "user".into()],
    };

    let mut req = req;
    req.extensions_mut().insert(user);
    next.run(req).await
}

let auth_state = AuthState { jwt_secret: Arc::new("secret".into()) };
let app = Router::new()
    .route("/me", get(|axum::extract::Extension(user): axum::extract::Extension<AuthenticatedUser>| async move {
        format!("user {} with roles {:?}", user.user_id, user.roles)
    }))
    .layer(middleware::from_fn_with_state(auth_state.clone(), jwt_auth))
    .with_state(auth_state);
```

### Rate Limiting Middleware

```rust
use axum::{middleware, Router, routing::get, http::StatusCode, extract::Extension};
use std::sync::Arc;
use std::collections::HashMap;
use tokio::sync::Mutex;
use std::time::{Instant, Duration};

#[derive(Clone)]
struct RateLimitState {
    // IP -> (count, window_start)
    buckets: Arc<Mutex<HashMap<String, (u32, Instant)>>>,
    max_requests: u32,
    window: Duration,
}

async fn rate_limit(
    Extension(state): Extension<RateLimitState>,
    req: axum::http::Request<axum::body::Body>,
    next: middleware::Next,
) -> impl axum::response::IntoResponse {
    let ip = req
        .headers()
        .get("x-forwarded-for")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("unknown")
        .to_string();

    let mut buckets = state.buckets.lock().await;
    let now = Instant::now();

    let (count, window_start) = buckets
        .entry(ip)
        .or_insert((0, now));

    // Reset window if expired
    if now.duration_since(*window_start) > state.window {
        *count = 0;
        *window_start = now;
    }

    if *count >= state.max_requests {
        return (StatusCode::TOO_MANY_REQUESTS, "rate limit exceeded").into_response();
    }

    *count += 1;
    drop(buckets);
    next.run(req).await
}

let rate_state = RateLimitState {
    buckets: Arc::new(Mutex::new(HashMap::new())),
    max_requests: 100,
    window: Duration::from_secs(60),
};

let app = Router::new()
    .route("/", get(|| async { "hello" }))
    .layer(axum::Extension(rate_state))
    .layer(middleware::from_fn(rate_limit));
```

### Request Timing Middleware

```rust
use axum::{middleware, Router, routing::get, http::HeaderName, HeaderValue};
use std::time::Instant;

async fn timing_middleware(
    req: axum::http::Request<axum::body::Body>,
    next: middleware::Next,
) -> impl axum::response::IntoResponse {
    let start = Instant::now();
    let method = req.method().clone();
    let path = req.uri().path().to_string();

    let mut response = next.run(req).await;

    let duration = start.elapsed();
    let ms = duration.as_secs_f64() * 1000.0;

    response.headers_mut().insert(
        HeaderName::from_static("x-response-time").unwrap(),
        HeaderValue::from_str(&format!("{:.2}ms", ms)).unwrap(),
    );

    tracing::info!(
        method = %method, path = %path, duration_ms = ms,
        status = response.status().as_u16(),
        "request"
    );

    response
}

let app = Router::new()
    .route("/", get(|| async { "hello" }))
    .layer(middleware::from_fn(timing_middleware));
```

### Body Size Check Middleware

```rust
use axum::{
    body::Body,
    http::{Request, StatusCode, header},
    middleware, response::IntoResponse, routing::post, Router,
};
use futures_util::StreamExt;

// Check body size before passing to handler.
// This runs BEFORE the handler consumes the body.
async fn limit_body_size(
    req: Request<Body>,
    next: middleware::Next,
) -> impl IntoResponse {
    let content_length: Option<usize> = req
        .headers()
        .get(header::CONTENT_LENGTH)
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.parse().ok());

    const MAX_SIZE: usize = 1024 * 1024; // 1 MB

    if let Some(len) = content_length {
        if len > MAX_SIZE {
            return (
                StatusCode::PAYLOAD_TOO_LARGE,
                format!("body too large: {} bytes, max {}", len, MAX_SIZE),
            ).into_response();
        }
    }

    next.run(req).await
}

let app = Router::new()
    .route("/upload", post(|| async { "uploaded" }))
    .layer(middleware::from_fn(limit_body_size));
```

> **Note**: For production body limiting, prefer `tower_http::limit::RequestBodyLimitLayer` which
> streams the body and checks size incrementally without buffering the entire body.

---

## Common Pitfalls

### 1. Middleware Ordering is Inverted with `.layer()`

Each `.layer()` call wraps the previous one. The **last** `.layer()` is the **outermost** — it runs
first on requests and last on responses.

```rust
// WRONG: expect auth before logging
.layer(middleware::from_fn(auth_mw))   // inner
.layer(middleware::from_fn(log_mw))    // outer — runs FIRST

// CORRECT: auth before logging (use ServiceBuilder or reverse order)
.layer(
    ServiceBuilder::new()
        .layer(middleware::from_fn(log_mw))   // outer
        .layer(middleware::from_fn(auth_mw))  // inner
)
```

**Rule of thumb**: Use `ServiceBuilder` and declare in the order you want execution.

### 2. Extracting the Request Body in Middleware

If your middleware extracts something that consumes the body, the handler receives an empty body.
Either avoid body extraction in middleware or manually reconstruct the request.

```rust
// BAD: consumes the body — handler gets nothing
async fn bad_mw(body: axum::body::Bytes, next: middleware::Next) -> impl IntoResponse {
    let size = body.len();
    // body is consumed here, cannot pass to handler
    // You must reconstruct the request or avoid consuming the body in middleware.
    // next.run(req) with an empty body is usually wrong.
    (axum::http::StatusCode::OK, format!("body size: {size}")).into_response()
}

// GOOD: inspect headers only, let body pass through
async fn good_mw(req: axum::http::Request<axum::body::Body>, next: middleware::Next) -> impl IntoResponse {
    let content_type = req.headers().get("content-type").cloned();
    let mut response = next.run(req).await;
    // ... optionally inspect response
    response
}
```

### 3. State Mismatch with `from_fn_with_state`

The state type passed to `from_fn_with_state` must match the router's state type. If you have
different state types for different middleware, use `Extension` instead.

```rust
// If middleware needs different data than the router state, use Extension:

#[derive(Clone)]
struct RateLimitConfig { limit: u32 }

let rate_config = RateLimitConfig { limit: 100 };

let app = Router::new()
    .route("/", get(|| async { "ok" }))
    .layer(axum::Extension(rate_config))
    .layer(middleware::from_fn(rate_limit_mw));  // extracts Extension<RateLimitConfig>
```

### 4. Double-Processing with `.route_layer()` and `.layer()`

Calling `.route_layer()` on a router and then `.layer()` on the merged result means routes get
**both** layers. Be explicit about which routes need which middleware.

```rust
// Every request to /admin hits admin_auth AND logging
Router::new()
    .route("/admin/data", get(admin_handler))
    .route_layer(middleware::from_fn(admin_auth))  // only /admin routes
    .merge(
        Router::new()
            .route("/public", get(public_handler))
    )
    .layer(middleware::from_fn(logging));  // ALL routes
```

### 5. Forgetting `IntoResponse` for Error Types

If your middleware returns `Result<T, E>`, both `T` and `E` must implement `IntoResponse`.
Otherwise you get confusing compiler errors about trait bounds.

```rust
// Compiler error: AuthError does not implement IntoResponse
async fn bad_mw(next: middleware::Next) -> Result<impl IntoResponse, AuthError> {
    Err(AuthError::Missing)  // won't compile without impl IntoResponse for AuthError
}

// Fix: implement IntoResponse (see Error Handling section above)
```

### 6. Async Block Captures in from_fn Closures

`from_fn` takes a function pointer, not a closure with captures. Use `from_fn_with_state` or
`Extension` to pass data, not closure captures.

```rust
// DOES NOT COMPILE: from_fn requires Fn, not FnOnce with captures
let secret = "my-secret".to_string();
.layer(middleware::from_fn(move |next| async move {
    // cannot use `secret` here
    next.run(()).await
}))

// CORRECT: use Extension or from_fn_with_state
.layer(axum::Extension(secret))
.layer(middleware::from_fn(|Extension(secret): Extension<String>, next: middleware::Next| async move {
    let _ = &secret;
    next.run(()).await
}))
```
