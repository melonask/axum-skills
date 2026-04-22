# Axum 0.8 Routing Reference

A deep guide to routing in axum 0.8.x. Every concept explains _why_ it behaves the way it does.

## 1. Basic Route Definition

Bind an HTTP method + path to a handler function using `.route()`. Axum builds on **matchit** (v0.8) for path matching and **tower** for the service layer — every handler is just a `Service<Request, Response = Response>`.

```rust
use axum::{Router, routing::{get, post, put, delete, patch, head, options}, Json};
use serde_json::{json, Value};

async fn health_check() -> &'static str { "ok" }
async fn create_item(Json(body): Json<Value>) -> Json<Value> { Json(json!({ "created": body })) }
async fn read_item() -> &'static str { "item" }
async fn update_item() -> &'static str { "updated" }
async fn delete_item() -> &'static str { "deleted" }
async fn patch_item() -> &'static str { "patched" }
async fn head_item() -> &'static str { "" }
async fn options_item() -> &'static str { "" }

let app = Router::new()
    .route("/health", get(health_check))
    .route("/items", post(create_item))
    .route("/items/{id}", get(read_item).put(update_item).delete(delete_item))
    .route("/items/{id}", patch(patch_item).head(head_item).options(options_item));
```

**Why**: Each call to `.route()` registers a _matchit_ node. The method-specific functions (`get`, `post`, …) return a `MethodRouter` that dispatches only matching HTTP verbs. If a request's method is registered, the handler runs; otherwise axum falls through to the fallback (default: 405).

## 2. Path Parameter Syntax (matchit 0.8)

Path segments wrapped in `{}` capture values. Use `Path<T>` extractor to deserialize them.

```rust
use axum::extract::Path;
use serde::Deserialize;

// Single parameter
async fn get_user(Path(id): Path<u32>) -> String {
    format!("user {id}")
}

// Multiple parameters — tuple order matches declaration order
async fn get_org_repo(Path((org, repo)): Path<(String, String)>) -> String {
    format!("{org}/{repo}")
}

// Regex constraint: only digits — {:param} syntax with a matchit pattern
// NOTE: axum 0.8 uses matchit 0.8 which supports {:id} for inline regex
async fn get_post(Path(id): Path<u32>) -> String {
    format!("post {id}")
}

let app = Router::new()
    .route("/users/{id}", get(get_user))
    .route("/repos/{org}/{repo}", get(get_org_repo))
    .route("/posts/{:id}", get(get_post));  // {:id} = regex-constrained (digits only by default)

// Wildcard catch-all — {*}path captures the rest of the URI including slashes
async fn catch_all(Path(path): Path<String>) -> String {
    format!("fallback path: {path}")
}
let app2 = Router::new()
    .route("/{*path}", get(catch_all));
// A request to GET /anything/deep/nested yields path = "anything/deep/nested"
```

**Why**: matchit compiles routes into a radix tree at build time. Named params (`{id}`) match a single segment; regex params (`{:id}`) restrict matches by pattern; catch-alls (`{*path}`) greedily consume all remaining segments. This enables O(k) lookup where k is the path depth.

## 3. MethodRouter Chaining

Chain multiple HTTP methods on the same path. `MethodRouter` is a builder — each method call adds a branch without overwriting previous ones.

```rust
use axum::{routing::{get, post, put, delete}, Json};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct CreateUser { name: String }

#[derive(Serialize)]
struct User { id: u64, name: String }

async fn list_users() -> Json<Vec<User>> { Json(vec![]) }
async fn create_user(Json(_input): Json<CreateUser>) -> Json<User> {
    Json(User { id: 1, name: "alice".into() })
}
async fn update_user() -> &'static str { "updated" }
async fn delete_user() -> &'static str { "deleted" }

let app = Router::new()
    .route(
        "/users",
        get(list_users).post(create_user),
    )
    .route(
        "/users/{id}",
        get(|| async { "user detail" })
            .put(update_user)
            .delete(delete_user),
    );
```

**Why**: A `MethodRouter` internally holds a `HashMap<Method, Handler>`. Calling `.get()`, `.post()`, etc. inserts into this map. When axum matches a route, it looks up the request method in this map — if absent, it triggers a 405 fallback, not a 404. This distinction matters for REST APIs where the resource exists but the method is unsupported.

## 4. Nested Routers

Use `.nest()` to mount a complete `Router` under a path prefix. The inner router's routes are resolved _relative_ to the prefix.

```rust
use axum::{Router, routing::get};

async fn api_status() -> &'static str { "v1 status" }
async fn list_products() -> &'static str { "products" }
async fn get_product(Path(id): Path<u64>) -> String { format!("product {id}") }

let products_router = Router::new()
    .route("/", get(list_products))
    .route("/{id}", get(get_product));

let v1_router = Router::new()
    .route("/status", get(api_status))
    .nest("/products", products_router);

let app = Router::new()
    .nest("/api/v1", v1_router);
// GET /api/v1/status          -> "v1 status"
// GET /api/v1/products        -> "products"
// GET /api/v1/products/42     -> "product 42"
```

**Why**: `.nest()` strips the prefix from the URI before passing it to the inner router's matchit tree. This lets teams develop sub-routers independently and compose them hierarchically. The prefix is consumed at the outer layer, so inner routes never see it.

## 5. Merge Routers

`.merge()` flattens another router's routes into the current one — no prefix is added. Routes from both routers live at the same level.

```rust
use axum::{Router, routing::get};

async fn users_list() -> &'static str { "users" }
async fn orders_list() -> &'static str { "orders" }

let users = Router::new().route("/users", get(users_list));
let orders = Router::new().route("/orders", get(orders_list));

let app = Router::new()
    .merge(users)
    .merge(orders);
// GET /users  -> "users"
// GET /orders -> "orders"
```

**Why**: Unlike `.nest()`, merge does _not_ introduce a path prefix — it clones every route node from the source into the target's matchit tree. Use merge when organizing routes across modules/files but want them all at the root level. Beware: if both routers define the same path, the last `.merge()` wins and panics at runtime if routes conflict.

## 6. Fallback

Handle requests that match no route (404) or match a path but not a method (405).

```rust
use axum::{Router, routing::get, http::StatusCode, response::{IntoResponse, Response}};

async fn not_found() -> Response {
    (StatusCode::NOT_FOUND, "Nothing here").into_response()
}

async fn method_not_allowed() -> Response {
    (StatusCode::METHOD_NOT_ALLOWED, "Try a different HTTP method").into_response()
}

let app = Router::new()
    .route("/items", get(|| async { "items" }))
    .fallback(not_found)
    .method_not_allowed_fallback(method_not_allowed);
// GET  /items     -> "items"
// POST /items     -> 405 "Try a different HTTP method"
// GET  /missing   -> 404 "Nothing here"
```

`.fallback_service()` accepts any `tower::Service` for full control:

```rust
use axum::body::Body;
use tower_http::services::ServeDir;

let app = Router::new()
    .route("/api", get(|| async { "api" }))
    .fallback_service(ServeDir::new("./static"));
// Requests to /index.html serve the file; /api still hits the handler.
```

**Why**: Axum's default fallback returns a plain 404/405. In production you typically want JSON error bodies or a SPA catch-all. The method-not-allowed fallback fires _only_ when matchit finds the path but the `MethodRouter` has no entry for the request method — this is how axum distinguishes "wrong path" from "wrong method".

## 7. Route Service & Nest Service

Mount arbitrary `tower::Service` implementations directly on specific routes.

```rust
use axum::Router;
use tower_http::services::{ServeDir, ServeFile};

let app = Router::new()
    // Serve a single file at a route
    .route_service("/favicon.ico", ServeFile::new("static/favicon.ico"))
    // Serve a directory tree
    .nest_service("/static", ServeDir::new("static/assets"))
    .route("/api/health", get(|| async { "ok" }));
// GET /favicon.ico       -> serves the file
// GET /static/app.js     -> serves static/assets/app.js
// GET /api/health        -> "ok"
```

**Why**: Regular handlers go through axum's extractor pipeline, but some services (file servers, gRPC transports, WebSocket upgraders) expect to consume the raw request. `route_service` and `nest_service` bypass extractors entirely, connecting the service directly into tower's middleware stack.

## 8. Dynamic Routes

Build routes programmatically — useful for plugins, feature flags, or code-generated APIs.

```rust
use axum::{Router, routing::get, extract::Path};
use std::collections::HashMap;

async fn dynamic_handler(Path((version, resource)): Path<(String, String)>) -> String {
    format!("v{version} resource: {resource}")
}

// Register the same handler under multiple version prefixes
let mut app = Router::new();
let versions = ["v1", "v2", "v3"];

for v in versions {
    let path = format!("/{v}/{{resource}}");
    app = app.route(&path, get(dynamic_handler));
}

// Conditional registration based on feature flags
#[cfg(feature = "admin")]
{
    app = app.route("/admin", get(|| async { "admin panel" }));
}
// GET /v2/products -> "v2 resource: products"
```

**Why**: axum's `Router` is a builder (it owns a matchit `Router<()>)`). Each `.route()` call returns a new `Router` with the route added. This makes it trivial to construct routes in loops or behind `cfg` gates. Note that matchit must be recompiled on each addition, so avoid building thousands of routes in tight loops at runtime.

## 9. Router Conversions

Convert a `Router` into a tower `Service` for use with hyper, `tower::ServiceExt`, or testing.

```rust
use axum::{Router, routing::get, body::Body};
use tower::ServiceExt; // for oneshot()

let app = Router::new().route("/", get(|| async { "hello" }));

// Clone the router as a Service without consuming it (for middleware / tests)
// as_service() — NOT available directly; use into_service() or clone the Service

// into_service() — consume the Router, get a Service
let service = app.clone().into_service();

// into_make_service() — wrap in a MakeService (required by hyper::Server)
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, app.into_make_service()).await.unwrap();

// into_make_service_with_connect_info() — expose the remote address to handlers
use axum::extract::ConnectInfo;
use std::net::SocketAddr;

async fn who_am_i(ConnectInfo(addr): ConnectInfo<SocketAddr>) -> String {
    format!("your IP: {addr}")
}

let app_with_addr = Router::new()
    .route("/", get(who_am_i))
    .into_make_service_with_connect_info::<SocketAddr>();
axum::serve(listener, app_with_addr).await.unwrap();
```

**Why**: hyper's `Server::serve` requires a `MakeService` that produces a new `Service` per connection. `into_make_service()` provides this. `into_make_service_with_connect_info()` wraps each incoming connection's socket address into request extensions so handlers can extract it via `ConnectInfo<T>`. For unit tests, `into_service()` + `ServiceExt::oneshot()` lets you send fake requests without starting a server.

## 10. Common Routing Patterns

### API Versioning with Nesting

```rust
mod v1 {
    use axum::{Router, routing::{get, post}, Json, extract::Path};
    use serde::{Deserialize, Serialize};

    #[derive(Serialize)]
    struct ApiStatus { version: &'static str }

    pub fn router() -> Router {
        Router::new()
            .route("/status", get(|| async { Json(ApiStatus { version: "v1" }) }))
            .route("/users/{id}", get(|Path(id): Path<u64>| async move { format!("v1 user {id}") }))
    }
}

let app = Router::new()
    .nest("/api/v1", v1::router());
```

### Health Check & Resource Grouping

```rust
use axum::{Router, routing::{get, post}, Json};
use serde_json::json;

let app = Router::new()
    // Infrastructure routes
    .route("/health", get(|| async { json!({ "status": "ok" }) }))
    .route("/ready", get(|| async { json!({ "ready": true }) }))
    // Resource group
    .route("/users", get(list_users).post(create_user))
    .route("/users/{id}", get(get_user).put(update_user).delete(delete_user))
    .route("/posts", get(list_posts).post(create_post));
```

### Modular Route Organization

```rust
fn create_app() -> Router {
    Router::new()
        .merge(health_routes())
        .nest("/api", api_routes())
        .nest_service("/assets", ServeDir::new("./public"))
        .fallback(not_found_handler)
}

fn health_routes() -> Router {
    Router::new()
        .route("/health", get(|| async { "ok" }))
        .route("/metrics", get(|| async { "..." }))
}

fn api_routes() -> Router {
    Router::new()
        .nest("/v1", v1::router())
}
```

## Common Pitfalls

1. **Trailing slashes matter.** `/users` and `/users/` are different nodes in matchit. If clients hit `/users/` but you registered `/users`, they get a 404 — not a redirect. Register both or use middleware to normalize.

2. **Merge vs. nest confusion.** `.merge()` flattens routes to the same level; `.nest()` adds a prefix. If you merge two routers that both define `/users`, the _second_ one silently overwrites or panics. Use `.nest()` when you want isolated namespaces.

3. **Catch-all greedy matching.** `{*path}` matches _everything_, including `/api/health`. Place catch-all routes last (or use `.fallback()`) so specific routes are checked first. matchit resolves by specificity, but a catch-all at a high-level prefix will shadow deeper routes.

4. **MethodRouter does not fall through to other `.route()` calls.** If you define `.route("/items", get(handler_a))` and later `.route("/items", post(handler_b))`, the second call _replaces_ the first for that path. Chain methods on a single `.route()` call instead: `.route("/items", get(a).post(b))`.

5. **`into_make_service_with_connect_info` requires the right type.** The type `T` must match what the listener produces. For TCP listeners use `SocketAddr`; for Unix sockets use `tokio::net::unix::SocketAddr`. Mismatched types cause a runtime panic when the first connection arrives.

6. **`nest` and path parameters interact.** `/api/{version}/users` nested under `/api` means the inner router sees `/{version}/users`. Design inner routers to be prefix-aware, or extract the prefix in middleware if you need it.
