# Tower-HTTP Layers Reference for Axum 0.8.x

Reference for composing tower-http middleware layers with axum 0.8.x.
All layers live under the `tower_http` crate. Each requires a feature flag in `Cargo.toml`.

## Cargo.toml Feature Flags

```toml
[dependencies]
tower-http = { version = "0.6", features = [
    "cors", "trace",
    "compression-gzip", "compression-br", "compression-deflate",
    "decompression-gzip", "decompression-br",
    "timeout", "limit", "fs", "add-extension",
    "request-id", "propagate-header", "sensitive-headers",
    "set-header", "set-status", "catch-panic",
    "normalize-path", "auth", "metrics",
] }
```

Include only the features you use. A missing flag causes `use of undeclared type` at compile time.

---

## 1. CorsLayer

Without CORS headers, browsers block cross-origin requests. `CorsLayer::permissive()` allows
everything (development only). Production requires restrictive mode with concrete origins,
methods, and headers. `allow_credentials(true)` requires concrete origins — browsers reject
`Any` with credentials. `max_age` caches the preflight result (seconds).

```toml
features = ["cors"]
```

```rust
use axum::{routing::get, Router};
use tower_http::cors::CorsLayer;
use http::{HeaderValue, Method};

let cors = CorsLayer::new()
    .allow_origin("https://myapp.com".parse::<HeaderValue>().unwrap())
    .allow_methods([Method::GET, Method::POST, Method::PUT])
    .allow_headers([http::header::CONTENT_TYPE, http::header::AUTHORIZATION])
    .expose_headers([http::header::AUTHORIZATION]) // visible to browser JS
    .allow_credentials(true)
    .max_age(std::time::Duration::from_secs(3600));

let app = Router::new().route("/api/data", get(|| async { "hello" })).layer(cors);
```

---

## 2. TraceLayer

Creates `tracing` spans for every HTTP request. `make_span_with` creates the span on arrival;
`on_response` and `on_failure` record fields into it so all log lines share the same context.
Use `TraceLayer::new_for_http()` without customization for sensible defaults.

```toml
features = ["trace"]
```

```rust
use axum::{routing::get, Router, http::Request, body::Body};
use tower_http::trace::TraceLayer;
use tracing::Span;

let app = Router::new().route("/", get(|| async { "ok" }))
    .layer(
        TraceLayer::new_for_http()
            .make_span_with(|req: &Request<Body>| {
                tracing::info_span!("http_request", method = %req.method(), uri = %req.uri(),
                    status = tracing::field::Empty, latency = tracing::field::Empty)
            })
            .on_request(|_req: &Request<Body>, _s: &Span| { tracing::info!("received"); })
            .on_response(|res: &axum::response::Response, lat: std::time::Duration, s: &Span| {
                s.record("status", res.status().as_u16());
                s.record("latency", format!("{:?}", lat));
            })
            .on_failure(|err: tower_http::classify::ServerErrorsFailureClass,
                         lat: std::time::Duration, s: &Span| {
                s.record("status", 500u16);
                tracing::error!("failed: {:?} in {:?}", err, lat);
            }),
    );
```

---

## 3. CompressionLayer / DecompressionLayer

Negotiates response compression via `Accept-Encoding`. Place it outermost so the final body is
compressed after all inner layers have run. `DecompressionLayer` auto-decompresses request bodies.

```toml
features = ["compression-gzip", "compression-br", "compression-deflate", "decompression-gzip"]
```

```rust
use axum::{routing::{get, post}, Router, extract::Bytes};
use tower_http::{compression::{CompressionLayer, compression::CompressionLevel},
                 decompression::DecompressionLayer};

let app = Router::new()
    .route("/large", get(|| async { vec![0u8; 100_000] }))
    .route("/upload", post(|body: Bytes| async { format!("got {} bytes", body.len()) }))
    .layer(DecompressionLayer::new())
    .layer(CompressionLayer::new().quality(CompressionLevel::Default));
```

---

## 4. TimeoutLayer

Aborts slow requests with a 408. Per-route timeouts override global ones because the innermost
layer fires first on the response path.

```toml
features = ["timeout"]
```

```rust
use axum::{routing::get, Router};
use tower_http::timeout::TimeoutLayer;
use std::time::Duration;

let app = Router::new()
    .route("/fast", get(|| async { "quick" }))
    .route("/slow", get(|| async {
        tokio::time::sleep(Duration::from_secs(5)).await; "ok"
    }).layer(TimeoutLayer::new(Duration::from_secs(10)))) // per-route: 10s
    .layer(TimeoutLayer::new(Duration::from_secs(3))); // global: 3s
```

---

## 5. RequestBodyLimitLayer

Rejects oversized request bodies before extractors run. `DefaultBodyLimit` operates at the
extractor level (defaults to 2 MB); `RequestBodyLimitLayer` rejects at the middleware level.
Use both for defense in depth.

```toml
features = ["limit"]
```

```rust
use axum::{routing::post, Router, extract::{Bytes, DefaultBodyLimit}};
use tower_http::limit::RequestBodyLimitLayer;

let app = Router::new()
    .route("/upload", post(|_body: Bytes| async { "ok" }))
    .layer(DefaultBodyLimit::max(5 * 1024 * 1024))
    .layer(RequestBodyLimitLayer::new(5 * 1024 * 1024));
```

---

## 6. ServeDir / ServeFile

Serves static files with MIME detection, ETag/304, and precompressed variant resolution.
Use `.not_found_service(ServeFile::new(...))` for SPA fallback on client-side routing.

```toml
features = ["fs"]
```

```rust
use axum::Router;
use tower_http::services::{ServeDir, ServeFile};

let app = Router::new()
    .nest_service("/static", ServeDir::new("assets"))
    .fallback_service(ServeDir::new("dist").not_found_service(ServeFile::new("dist/index.html")));
```

---

## 7. AddExtensionLayer

Injects data into request extensions, extractable via `Extension<T>`. Prefer axum's `State` for
app-level state; use this when tower middleware needs its own data injected.

```toml
features = ["add-extension"]
```

```rust
use axum::{routing::get, Router, extract::Extension};
use tower_http::add_extension::AddExtensionLayer;
use std::sync::Arc;

#[derive(Clone)]
struct AppState { db_pool: Arc<String> }

async fn handler(Extension(state): Extension<Arc<AppState>>) -> String {
    format!("pool: {:?}", state.db_pool)
}

let app = Router::new().route("/", get(handler))
    .layer(AddExtensionLayer::new(Arc::new(AppState {
        db_pool: Arc::new("connected".into()),
    })));
```

---

## 8. SetRequestIdLayer

Generates a unique ID per request as `X-Request-ID` on both request and response. Correlate in
tracing spans to link all logs for a single request.

```toml
features = ["request-id"]
```

```rust
use axum::{routing::get, Router};
use tower_http::request_id::{MakeRequestUuid, SetRequestIdLayer};
use http::header::HeaderName;

let app = Router::new().route("/", get(|| async { "hello" }))
    .layer(SetRequestIdLayer::new(
        HeaderName::from_static("x-request-id"), MakeRequestUuid));
```

---

## 9. PropagateHeaderLayer

Copies a header from the request to the response. Combine with `SetRequestIdLayer` so the
generated ID flows: request → handler → response header visible to client.

```toml
features = ["propagate-header"]
```

```rust
use axum::{routing::get, Router};
use tower_http::propagate_header::PropagateHeaderLayer;
use http::header::HeaderName;

let app = Router::new().route("/", get(|| async { "hello" }))
    .layer(PropagateHeaderLayer::new(HeaderName::from_static("x-request-id")));
```

---

## 10. SensitiveHeaderLayer

Masks header values in trace logs. Place it **outside** `TraceLayer` (before in `.layer()` chain)
so it marks headers sensitive before TraceLayer logs them.

```toml
features = ["sensitive-headers"]
```

```rust
use axum::{routing::get, Router};
use tower_http::{trace::TraceLayer, sensitive_headers::SetSensitiveHeadersLayer};
use http::header::HeaderName;

let app = Router::new().route("/", get(|| async { "ok" }))
    .layer(TraceLayer::new_for_http())
    .layer(SetSensitiveHeadersLayer::new([
        HeaderName::from_static("authorization"),
        HeaderName::from_static("cookie"),
    ]));
```

---

## 11. SetHeaderLayer / SetStatusLayer

Overwrites response headers or status codes. `overriding` always sets; `if_not_present` avoids
clobbering handler-set headers.

```toml
features = ["set-header", "set-status"]
```

```rust
use axum::{routing::get, Router};
use tower_http::{set_header::SetHeaderLayer, set_status::SetStatusLayer};
use http::{HeaderValue, header, StatusCode};

let app = Router::new().route("/", get(|| async { "hello" }))
    .layer(SetHeaderLayer::overriding(header::X_FRAME_OPTIONS, HeaderValue::from_static("DENY")))
    .layer(SetHeaderLayer::if_not_present(header::SERVER, HeaderValue::from_static("my-app")))
    .layer(SetStatusLayer::new(StatusCode::CREATED));
```

---

## 12. CatchPanicLayer

Converts handler panics into 500 responses. Without it, a panic crashes the task and the client
gets a connection reset. Place outermost to catch panics from all inner layers.

```toml
features = ["catch-panic"]
```

```rust
use axum::{routing::get, Router};
use tower_http::catch_panic::CatchPanicLayer;

let app = Router::new().route("/panic", get(|| async { panic!("crash") }))
    .layer(CatchPanicLayer::new());
```

---

## 13. NormalizePathLayer

Normalizes paths (removes trailing slashes, collapses `//`) so `/api/users/`, `/api/users//`,
and `/api/users` route to the same handler. Issues a 308 redirect by default.

```toml
features = ["normalize-path"]
```

```rust
use axum::{routing::get, Router};
use tower_http::normalize_path::NormalizePathLayer;

let app = Router::new().route("/api/users", get(|| async { "users" }))
    .layer(NormalizePathLayer::trim_trailing_slash());
```

---

## 14. RequireAuthorizationLayer

Enforces HTTP auth before requests reach handlers, rejecting with 401 immediately.
Place on specific route groups so public routes remain accessible.

```toml
features = ["auth"]
```

```rust
use axum::{routing::get, Router, response::IntoResponse};
use tower_http::auth::RequireAuthorizationLayer;
use http::{HeaderValue, StatusCode};

// Basic auth with custom rejection
let basic = Router::new().route("/protected", get(|| async { "secret" }))
    .layer(RequireAuthorizationLayer::basic("admin", "secret")
        .on_unauthorized(StatusCode::FORBIDDEN.into_response()));

// Bearer token auth
let bearer = Router::new().route("/api", get(|| async { "protected" }))
    .layer(RequireAuthorizationLayer::bearer(HeaderValue::from_static("my-secret-token")));
```

---

## 15. MetricsLayer

Exports Prometheus metrics (`http_requests_total`, `http_requests_duration_seconds`). No handler
changes needed. Serve `/metrics` via `PrometheusBuilder` or a dedicated route.

```toml
features = ["metrics"]
metrics = "0.24"
metrics-exporter-prometheus = "0.16"
```

```rust
use axum::{routing::get, Router};
use tower_http::metrics::MetricsLayer;
use metrics_exporter_prometheus::PrometheusBuilder;

PrometheusBuilder::new().install()?;

let app = Router::new().route("/", get(|| async { "hello" }))
    .layer(MetricsLayer::new());
```

---

## 16. ServiceBuilder — Composing Multiple Layers

Use `tower::ServiceBuilder` to compose layers. First listed = outermost (closest to client),
last listed = innermost (closest to handler). Layers apply inside-out on the request path and
outside-in on the response path.

```rust
use axum::{routing::{get, post}, Router, extract::Extension};
use tower::ServiceBuilder;
use tower_http::{
    cors::CorsLayer, trace::TraceLayer, compression::CompressionLayer,
    timeout::TimeoutLayer, catch_panic::CatchPanicLayer,
    request_id::{MakeRequestUuid, SetRequestIdLayer},
    propagate_header::PropagateHeaderLayer,
    sensitive_headers::SetSensitiveHeadersLayer,
    limit::RequestBodyLimitLayer, add_extension::AddExtensionLayer,
};
use http::header::HeaderName;
use std::time::Duration;

#[derive(Clone)] struct AppState { db: String }

let app = Router::new()
    .route("/", get(|| async { "root" }))
    .route("/api/data", post(|Extension(s): Extension<AppState>| async move { s.db }))
    .layer(ServiceBuilder::new()
        // 1 (outermost) — CORS on all responses including errors
        .layer(CorsLayer::permissive())
        // 2 — echo X-Request-ID to client
        .layer(PropagateHeaderLayer::new(HeaderName::from_static("x-request-id")))
        // 3 — generate request ID
        .layer(SetRequestIdLayer::new(HeaderName::from_static("x-request-id"), MakeRequestUuid))
        // 4 — tracing (needs request ID)
        .layer(TraceLayer::new_for_http())
        // 5 — mask secrets before trace logs
        .layer(SetSensitiveHeadersLayer::new([HeaderName::from_static("authorization")]))
        // 6 — compress response
        .layer(CompressionLayer::new())
        // 7 — global timeout
        .layer(TimeoutLayer::new(Duration::from_secs(30)))
        // 8 — body size limit
        .layer(RequestBodyLimitLayer::new(10 * 1024 * 1024))
        // 9 (innermost) — catch panics
        .layer(CatchPanicLayer::new()))
    .layer(AddExtensionLayer::new(AppState { db: "connected".into() }));
```

---

## Common Pitfalls

**Layer ordering:** `CompressionLayer` must see the final uncompressed body, so it goes outside.
`CatchPanicLayer` must wrap everything. `CorsLayer` must be outermost — browsers need CORS
headers on error responses (401, 500). The response path flows from innermost to outermost.

**Per-route vs global:** `.layer()` on `Router` is global; `.layer()` on a handler is per-route.
Per-route layers execute _inside_ global layers, so a per-route `TimeoutLayer` fires first on the
response path and overrides the global timeout.

**Extensions and ordering:** `AddExtensionLayer` must wrap a handler (be outside it) for
`Extension<T>` extraction to work. Use axum's `State` to bypass this constraint entirely.

**Missing features:** Each module requires its own feature flag. Match `Cargo.toml` features to
your `use` statements.
