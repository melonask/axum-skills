# Axum State Management Reference

Guide for axum 0.8.x. All examples are complete and runnable.

## 1. Router::with_state — Finalizing a Router

`Router::with_state(state)` converts `Router<S>` into `Router<()>`. Call it exactly once at the top level after combining all sub-routers. `axum::serve` only accepts `Router<()>` — state has already been captured by that point.

```rust
use axum::{routing::get, Router};
use std::sync::Arc;

#[derive(Clone)]
struct AppState { db_pool: sqlx::PgPool }

let state = Arc::new(AppState { /* ... */ });
let app: Router<()> = Router::new()
    .route("/users", get(list_users))
    .with_state(state);
```

**Why `Clone + Send + Sync`?** Axum clones state per request (via the `State<T>` extractor). `Send + Sync` is required because Tokio's multi-threaded scheduler may move request handlers between threads.

## 2. State<T> Extractor — Per-Request Cloning

`State<T>` is an extractor. Add it as a handler parameter and axum clones the app state automatically. When `T` wraps data in `Arc`, cloning is zero-cost (only bumps a reference count).

```rust
use axum::{extract::State, Json};
use serde_json::{json, Value};

#[derive(Clone)]
struct AppState { db_pool: sqlx::PgPool }

async fn list_users(State(state): State<Arc<AppState>>) -> Json<Value> {
    let users = sqlx::query_as!(User, "SELECT id, name FROM users")
        .fetch_all(&state.db_pool).await.unwrap();
    Json(json!({ "users": users }))
}
```

**Why cloning?** Axum uses clone-on-extract so state stays independent of any single request's lifetime. Multiple requests run concurrently without holding a borrow on the original.

## 3. FromRef Trait — Extracting Sub-Parts

Use `FromRef` when handlers need only a portion of the app state (e.g., just the DB pool). This keeps handler signatures focused and decouples handlers from the top-level state shape.

```rust
use axum::extract::FromRef;

#[derive(Clone)]
struct AppState { db_pool: sqlx::PgPool, redis: redis::aio::ConnectionManager }

impl FromRef<AppState> for sqlx::PgPool {
    fn from_ref(state: &AppState) -> Self { state.db_pool.clone() }
}

// Handlers can now accept State<sqlx::PgPool> directly
async fn health_check(State(pool): State<sqlx::PgPool>) -> &'static str {
    if sqlx::query("SELECT 1").execute(&pool).await.is_ok() { "ok" } else { "unhealthy" }
}
```

**Why FromRef?** If you restructure `AppState`, only the `FromRef` impls change — individual handlers stay untouched.

## 4. #[derive(FromRef)] — Auto-Generating Implementations

The `axum_macros::FromRef` derive generates `FromRef` impls for every field automatically. Every field type must implement `Clone`.

```rust
use axum::extract::FromRef;
use axum_macros::FromRef;

#[derive(Clone)]
struct Config { jwt_secret: String }

#[derive(Clone, FromRef)]
struct AppState {
    db_pool: sqlx::PgPool,
    redis: redis::aio::ConnectionManager,
    config: Arc<Config>,
}
// Generates: FromRef<AppState> for sqlx::PgPool, ConnectionManager, Arc<Config>

async fn get_config(State(config): State<Arc<Config>>) -> String {
    config.jwt_secret.clone()
}
```

If a field does not implement `Clone`, wrap it in `Arc` or write the impl manually.

## 5. Extension<T> — Runtime-Injected Per-Request Data

`Extension` injects data via Tower middleware, unlike `State` which is set at router construction. Use `Extension` for per-request data injected by middleware (auth context, request IDs). Never put mutable shared state in `Extension` — it is not shared across requests.

```rust
use axum::{extract::Extension, middleware, Router, routing::get};

#[derive(Clone)]
struct CurrentUser { user_id: i64, roles: Vec<String> }

async fn require_auth(req: axum::extract::Request, next: middleware::Next) -> axum::response::Response {
    let user = CurrentUser { user_id: 42, roles: vec!["admin".into()] };
    req.extensions().insert(user);
    next.run(req).await
}

async fn me(Extension(user): Extension<CurrentUser>) -> String {
    format!("user_id={}, roles={:?}", user.user_id, user.roles)
}

let app = Router::new().route("/me", get(me)).layer(middleware::from_fn(require_auth));
```

**Extension vs State:** Use `State` for application-wide server-lifetime data. Use `Extension` for per-request data injected by middleware.

## 6. State Sharing Patterns

### Arc<T> — Shared Read-Only Data

```rust
#[derive(Clone)]
struct AppState { config: Arc<Config> } // immutable after init
```

### Arc<RwLock<T>> — Shared Read-Write Data

Use `tokio::sync::RwLock` in async code. `std::sync::RwLock` uses OS-level blocking — holding a write guard across `.await` blocks the entire Tokio worker thread. `tokio::sync::RwLock` yields the task instead.

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

#[derive(Clone)]
struct AppState { counter: Arc<RwLock<i64>> }

async fn increment(State(state): State<AppState>) -> String {
    let mut guard = state.counter.write().await; // async — does not block OS thread
    *guard += 1;
    format!("count = {}", *guard)
}
```

### Arc<Mutex<T>> — Exclusive Access

Use `tokio::sync::Mutex` in async contexts for the same reason.

```rust
#[derive(Clone)]
struct AppState { task_queue: Arc<tokio::sync::Mutex<VecDeque<Job>>> }
```

### Arc<DashMap<K, V>> — Concurrent Hash Map

`DashMap` shards internally so concurrent reads/writes do not contend on a single lock. Ideal for caches.

```rust
use dashmap::DashMap;
use std::sync::Arc;

#[derive(Clone)]
struct AppState { cache: Arc<DashMap<String, String>> }

async fn get_cached(State(state): State<AppState>) -> Option<String> {
    state.cache.get(&"key".to_string()).map(|r| r.value().clone())
}
```

## 7. State Lifetime Patterns

State lives as long as the `Router<()>` passed to `axum::serve`. Shutdown drops the listener, router, then state. Implement `Drop` for cleanup.

```rust
#[derive(Clone)]
struct AppState { db_pool: sqlx::PgPool }

impl Drop for AppState {
    fn drop(&mut self) {
        eprintln!("AppState dropped — closing connections");
        // PgPool closes connections on drop automatically
    }
}
```

## 8. Multiple State Types via FromRef

Combine `FromRef` with nested sub-routers so each group extracts only what it needs.

```rust
use axum::{extract::FromRef, routing::get, Router};
use axum_macros::FromRef;

#[derive(Clone, FromRef)]
struct AppState {
    db: sqlx::PgPool,
    cache: Arc<DashMap<String, Vec<u8>>>,
}

fn user_routes() -> Router<AppState> {
    Router::new().route("/users", get(|| async { "users" }))
    // Handlers use State<sqlx::PgPool> via FromRef
}
fn cache_routes() -> Router<AppState> {
    Router::new().route("/cache/:key", get(|| async { "cached" }))
    // Handlers use State<Arc<DashMap<...>>> via FromRef
}

let app = Router::new()
    .merge(user_routes())
    .merge(cache_routes())
    .with_state(AppState { /* ... */ });
```

## 9. State in Middleware

Use `middleware::from_fn_with_state` to pass state into middleware. State is cloned into the closure identically to handlers.

```rust
use axum::{extract::State, middleware::{self, Next}, response::Response, routing::get, Router};

#[derive(Clone)]
struct AppState { api_keys: Arc<HashSet<String>> }

async fn auth_middleware(
    State(state): State<Arc<AppState>>, req: axum::extract::Request, next: Next,
) -> Response {
    let key = req.headers().get("x-api-key").and_then(|v| v.to_str().ok());
    match key {
        Some(k) if state.api_keys.contains(k) => next.run(req).await,
        _ => axum::http::StatusCode::UNAUTHORIZED.into_response(),
    }
}

let app = Router::new()
    .route("/protected", get(|| async { "secret" }))
    .layer(middleware::from_fn_with_state(state, auth_middleware));
```

## 10. State Initialization — Build Before Serving

Construct state fully before starting the server. This ensures the app fails fast if DB connections or config parsing fail, rather than returning 500 on the first request.

```rust
use axum::{routing::get, Router};
use std::sync::Arc;

#[derive(Clone)]
struct AppState { db: sqlx::PgPool, config: Arc<Config> }

#[derive(Clone)]
struct Config { database_url: String }

async fn build_state() -> Result<AppState, Box<dyn std::error::Error>> {
    let database_url = std::env::var("DATABASE_URL")?;
    let db = sqlx::postgres::PgPoolOptions::new()
        .max_connections(5).connect(&database_url).await?;
    sqlx::migrate!("./migrations").run(&db).await?;
    Ok(AppState { db, config: Arc::new(Config { database_url }) })
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let state = build_state().await?;
    let app = Router::new().route("/health", get(|| async { "ok" })).with_state(state);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    axum::serve(listener, app).await?;
    Ok(())
}
```

## 11. Common Patterns

### Database Pool as State

```rust
async fn create_user(
    State(pool): State<sqlx::PgPool>, Json(input): Json<CreateUser>,
) -> Result<Json<User>, StatusCode> {
    let user = sqlx::query_as!(User, "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *",
        input.name, input.email)
        .fetch_one(&pool).await.map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok(Json(user))
}
```

### Redis Client as State

```rust
use redis::aio::ConnectionManager;

#[derive(Clone)]
struct AppState { redis: ConnectionManager }

async fn get_session(
    State(redis): State<ConnectionManager>,
    axum::extract::Path(token): axum::extract::Path<String>,
) -> String {
    let val: Option<String> = redis::cmd("GET")
        .arg(format!("session:{}", token))
        .query_async(&mut redis.clone()).await.unwrap();
    val.unwrap_or_default()
}
```

### Configuration as State

```rust
#[derive(Clone, serde::Deserialize)]
struct Config { database_url: String, jwt_secret: String, port: u16 }

impl Config {
    fn from_env() -> Result<Self, envy::Error> { envy::from_env() }
}
```

### App-Wide Metrics as State

```rust
use std::sync::atomic::{AtomicU64, Ordering};

#[derive(Clone)]
struct AppState { request_count: Arc<AtomicU64> }

async fn count_middleware(
    State(state): State<AppState>, req: axum::extract::Request, next: middleware::Next,
) -> Response {
    state.request_count.fetch_add(1, Ordering::Relaxed);
    next.run(req).await
}
```

## Common Pitfalls

1. **Forgetting `Clone` on state:** The compiler error is usually "the trait bound `MyState: Clone` is not satisfied". Add `#[derive(Clone)]` or wrap in `Arc`.

2. **Using `std::sync::Mutex` across `.await`:** This panics. Always use `tokio::sync::Mutex` in async code.

3. **Calling `with_state` twice:** `Router::with_state` consumes `Router<S>` and returns `Router<()>`. Calling it again on `Router<()>` silently produces a router with no state. Merge all sub-routers first, then call `with_state` once.

4. **Putting mutable shared data in `Extension`:** `Extension` values are per-request, not shared across requests. Use `State` with interior mutability (`Arc<RwLock<T>>`, `AtomicU64`, `DashMap`).

5. **Large state structs without `Arc`:** Cloning a struct with large `Vec` fields per request is expensive. Wrap shared fields in `Arc` so only pointer-sized clones occur.

6. **Deadlocks with `RwLock`:** Taking a read lock then upgrading to a write lock within the same task can deadlock. Keep lock scopes short and avoid holding locks across `.await` points.
