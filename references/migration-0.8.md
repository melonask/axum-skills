# Migrating from axum 0.7 to 0.8

Complete reference for upgrading every breaking change. Each section explains **why** the change happened and provides before/after examples.

---

## 1. Path Parameter Syntax

The colon-based syntax `/:id` conflicted with URI templates in other specs and made it impossible to distinguish parameters from literal segments. Axum 0.8 adopts the `matchit` 0.8 convention using braces.

**Before (0.7):**

```rust
let app = Router::new()
    .route("/users/:id", get(get_user))
    .route("/files/*path", get(serve_file))
    .route("/posts/:id:\\d+", get(get_post_by_number));

async fn get_user(Path(id): Path<u32>) { /* ... */ }
async fn serve_file(Path(path): Path<String>) { /* ... */ }
```

**After (0.8):**

```rust
let app = Router::new()
    .route("/users/{id}", get(get_user))
    .route("/files/{*path}", get(serve_file))
    // Regex constraints use a leading colon before the param name
    .route("/posts/{:id}", get(get_post_by_number));
```

Search-and-replace `/:(\w+)` with `{$1}` and `/\*(\w+)` with `{*$1}`. Adjust regex constraints from `{param:pattern}` to `{:param}`.

---

## 2. Host Extractor Moved to `axum-extra`

`Host` extraction depends on `http` header types that live outside axum's minimal dependency footprint. Moving it to `axum-extra` keeps the core crate lean.

**Before (0.7):**

```rust
use axum::extract::Host;

async fn handler(Host(host): Host) -> String {
    format!("host: {host}")
}
```

**After (0.8):**

```rust
use axum_extra::extract::Host;

async fn handler(Host(host): Host) -> String {
    format!("host: {host}")
}
```

Add `axum-extra` to `Cargo.toml` with the `host` feature:

```toml
axum-extra = { version = "0.10", features = ["host"] }
```

---

## 3. WebSocket Message Types Changed

Allocating a new `String` or `Vec<u8>` on every message created unnecessary heap pressure. The new types use reference-counted bytes that can be cloned cheaply and shared across tasks.

**Before (0.7):**

```rust
use axum::extract::ws::Message;

while let Some(Ok(msg)) = socket.recv().await {
    match msg {
        Message::Text(text) => println!("{text}"),       // String
        Message::Binary(data) => println!("{data:?}"),   // Vec<u8>
        _ => {}
    }
}
```

**After (0.8):**

```rust
use axum::extract::ws::Message;

while let Some(Ok(msg)) = socket.recv().await {
    match msg {
        Message::Text(text) => println!("{text}"),       // Utf8Bytes
        Message::Binary(data) => println!("{data:?}"),   // Bytes
        _ => {}
    }
}
```

Convert with `.to_string()` / `.to_vec()` or construct directly:

```rust
let msg = Message::Text("hello".into());   // &str -> Utf8Bytes
let msg = Message::Binary(vec![1, 2, 3].into());  // Vec<u8> -> Bytes
```

---

## 4. `WebSocket::close()` Removed

The implicit `close()` method made it impossible to send custom close codes or reasons. Sending close frames is now explicit via `Message::Close`.

**Before (0.7):**

```rust
socket.close().await?;
```

**After (0.8):**

```rust
// Normal close
socket.send(Message::Close(None)).await?;

// Close with code and reason
socket.send(Message::Close(Some(axum::extract::ws::CloseFrame {
    code: axum::extract::ws::close_code::NORMAL,
    reason: "bye".into(),
}))).await?;
```

Dropping the sender also sends a close frame automatically, so simply letting `socket` go out of scope is often sufficient.

---

## 5. `Option<T>` Extractor Behavior

In 0.7, `Option<Path<T>>` silently returned `None` on parse failure, hiding bugs. Now extractors must implement `OptionalFromRequestParts` or `OptionalFromRequest`. Only extractors designed for absence (headers, query params) return `None`; others reject the request.

**Before (0.7) — silently swallowed parse errors:**

```rust
async fn handler(id: Option<Path<u32>>) {
    // If the segment was "abc", id would be None — no error logged
}
```

**After (0.8) — rejects on parse failure:**

```rust
// Option<Path<T>> now rejects if the parameter exists but fails to parse.
// Use a String parameter and parse manually if you need soft failure:
async fn handler(id: Option<Path<String>>) {
    if let Some(Path(id)) = id {
        if let Ok(num) = id.parse::<u32>() {
            // use num
        }
    }
}
```

New optional extractors (`Json`, `Extension`, `Query`, `TypedHeader`) correctly return `None` when the item is genuinely absent.

---

## 6. Sync Requirement — Handlers Must Be `Send + Sync`

Axum now requires all handler futures to be `Send`. This is enforced by tower's `Service` trait. Remove `Rc`, `RefCell`, and other non-thread-safe types from handlers and state.

**Before (0.7) — could compile with Rc in some setups:**

```rust
use std::rc::Rc;
use std::cell::RefCell;

async fn handler(State(state): State<Rc<RefCell<AppState>>>) {
    state.borrow_mut().count += 1;
}
```

**After (0.8) — must use thread-safe types:**

```rust
use std::sync::Arc;
use std::sync::Mutex;

async fn handler(State(state): State<Arc<Mutex<AppState>>>) {
    state.lock().unwrap().count += 1;
}

// Better: use tokio::sync::Mutex for async-friendly locking
use tokio::sync::Mutex;

async fn handler(State(state): State<Arc<Mutex<AppState>>>) {
    let mut app = state.lock().await;
    app.count += 1;
}
```

Run `cargo check` and fix any "cannot be sent between threads safely" errors.

---

## 7. `serve()` API Changes

The `serve` function is now generic over the listener and IO types, enabling hyper 1.x flexibility. `tcp_nodelay` is no longer a method on `Serve` because the listener itself controls that setting.

**Before (0.7):**

```rust
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
axum::serve(listener, app)
    .tcp_nodelay(true)
    .await?;
```

**After (0.8):**

```rust
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
// Set nodelay on the listener socket itself before passing to serve
let listener = tokio::net::TcpListener::from_std(
    listener.into_std()?
).await?;

// Or more simply — just call serve directly:
axum::serve(listener, app).await?;
```

The returned `Serve` type now implements `tower::Service<Incoming>` for composability with middleware layers.

---

## 8. Removed APIs

| Removed Item                                  | Replacement                                                                         |
| --------------------------------------------- | ----------------------------------------------------------------------------------- |
| `axum::extract::Host`                         | `axum_extra::extract::Host`                                                         |
| `WebSocket::close()`                          | `socket.send(Message::Close(...)).await`                                            |
| `axum::extract::ws::Message::Text(String)`    | `Message::Text(Utf8Bytes)`                                                          |
| `axum::extract::ws::Message::Binary(Vec<u8>)` | `Message::Binary(Bytes)`                                                            |
| `axum::body::Body::from_stream`               | Use `axum::body::Body::from_stream` (still available but now wraps `http-body` 1.x) |
| `Serve::tcp_nodelay`                          | Configure on the listener before calling `serve`                                    |
| `Route::new()` (legacy)                       | Use `Router::new()` consistently                                                    |
| `axum::extract::ConnectInfo` as-is            | Migrate to `axum::extract::ConnectInfo<SocketAddr>` with explicit listener binding  |

Review compiler errors after upgrading; each removal surfaces as a clear error message.

---

## 9. New APIs

### Method Not Allowed Fallback

```rust
let app = Router::new()
    .route("/users", get(list_users).post(create_user))
    .method_not_allowed_fallback(method_not_allowed_handler);
```

### `NoContent` Response

```rust
use axum::response::NoContent;

async fn delete_user() -> NoContent {
    NoContent
}
```

### WebSocket over HTTP/2

WebSocket connections now work over HTTP/2 streams when the client and server both support it. No code change required — it is automatic.

### CONNECT Method

```rust
use axum::http::Method;

let app = Router::new()
    .route("/tunnel", any(tunnel_handler))
    .allow_method(Method::CONNECT);
```

### WebSocket Upgrade Protocol Selection

```rust
use axum::extract::ws::{WebSocketUpgrade, WebSocket};
use futures_util::SinkExt;

async fn ws_handler(ws: WebSocketUpgrade) -> impl IntoResponse {
    ws.protocols(["graphql-ws", "ws"])
        .on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    // socket.protocol() returns the negotiated protocol
}
```

### Router Fallback Reset

```rust
let app = Router::new()
    .route("/api/*path", any(api_handler))
    .reset_fallback(); // Clears any previously set fallback
```

### SSE Binary Data

Server-sent events now support binary event data via `Event::data(Bytes)` in addition to `Event::data(String)`.

---

## 10. `impl OptionalFromRequest` for Json and Extension

Standard library extractors now support optional extraction, returning `None` instead of rejecting when the payload is absent.

**Before (0.7) — needed a custom wrapper or middleware:**

```rust
// Workaround: use Result<Json<T>, _> or a custom extractor
```

**After (0.8) — use Option directly:**

```rust
use axum::extract::Json;
use serde::Deserialize;

#[derive(Deserialize)]
struct PatchPayload { name: Option<String> }

async fn patch(body: Option<Json<PatchPayload>>) -> impl IntoResponse {
    if let Some(Json(payload)) = body {
        // apply partial update
    }
    // body was empty — no-op
}

// Extension also works optionally:
async fn handler(ext: Option<Extension<MyType>>) -> impl IntoResponse {
    if let Some(Extension(val)) = ext {
        // use val
    }
}
```

---

## 11. `#[async_trait]` No Longer Required

Custom extractors and middleware implement tower traits with native `async fn` in traits thanks to RPITIT (Return Position Impl Trait In Traits). Remove `async-trait` from your extractor definitions.

**Before (0.7):**

```rust
use async_trait::async_trait;
use axum::extract::FromRequestParts;

#[async_trait]
impl<S> FromRequestParts<S> for MyExtractor
where S: Send + Sync,
{
    type Rejection = StatusCode;

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        // ...
    }
}
```

**After (0.8):**

```rust
use axum::extract::FromRequestParts;
// No #[async_trait] needed

impl<S: Send + Sync> FromRequestParts<S> for MyExtractor {
    type Rejection = StatusCode;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection> {
        // ...
    }
}
```

Remove `async-trait` from `Cargo.toml` if no other code depends on it.

---

## 12. Cargo.toml Changes

Update dependency versions and enable new feature flags.

**Before (0.7):**

```toml
[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
tower = "0.4"
tower-http = { version = "0.5", features = ["cors", "trace"] }
```

**After (0.8):**

```toml
[dependencies]
axum = { version = "0.8", features = ["ws"] }
axum-extra = { version = "0.10", features = ["typed-header", "host"] }
tokio = { version = "1", features = ["full"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["cors", "trace"] }
```

Key notes:

- `tower` upgraded to **0.5** (check for breaking changes in custom middleware).
- `tower-http` upgraded to **0.6** (review `CorsLayer`, `TraceLayer` API changes).
- The `ws` feature is still required for WebSocket support.
- Add `axum-extra` if you use `Host`, `Cookie`, `Query` with serde, or other extended extractors.

---

## Migration Checklist

- [ ] Update `Cargo.toml` dependency versions (axum 0.8, tower 0.5, tower-http 0.6)
- [ ] Add `axum-extra` with needed features (`host`, `typed-header`, etc.)
- [ ] Run `cargo check` and resolve compiler errors
- [ ] Replace `/:param` path syntax with `{param}` in all route definitions
- [ ] Replace `/*path` catch-all syntax with `{*path}`
- [ ] Adjust regex path constraints from `{param:\\d+}` to `{:param}`
- [ ] Move `use axum::extract::Host` to `use axum_extra::extract::Host`
- [ ] Update `Message::Text` handling from `String` to `Utf8Bytes`
- [ ] Update `Message::Binary` handling from `Vec<u8>` to `Bytes`
- [ ] Replace all `socket.close()` calls with explicit `Message::Close` sends
- [ ] Audit `Option<T>` extractors: ensure parse failures are handled correctly
- [ ] Remove `Rc`/`RefCell` from handlers and state; use `Arc`/`Mutex` instead
- [ ] Remove `.tcp_nodelay()` calls from `serve()` chains
- [ ] Remove `#[async_trait]` from custom extractors and middleware
- [ ] Remove `async-trait` dependency if unused elsewhere
- [ ] Verify tower 0.5 / tower-http 0.6 compatibility for custom layers
- [ ] Run full test suite and fix any remaining failures
- [ ] Review the [axum 0.8 changelog](https://github.com/tokio-rs/axum/releases) for edge cases
