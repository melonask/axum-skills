# Axum Real-Time Communication Reference (WebSocket & SSE)

Targets **axum 0.8.x**. All examples use `tokio` and are fully runnable.

---

## 1. WebSocket Setup

Use the `WebSocketUpgrade` extractor to accept a WebSocket handshake. Call `on_upgrade` with an async handler that receives the raw `WebSocket` stream. Register the route with `.any()` (not `.get()`) so that both HTTP/1.1 and HTTP/2 clients can connect — HTTP/2 WebSocket upgrades arrive as POST requests in axum.

```rust
use axum::{
    extract::ws::{WebSocket, WebSocketUpgrade, Message},
    routing::any,
    Router,
};
use futures_util::{SinkExt, StreamExt};

async fn ws_handler(ws: WebSocketUpgrade) -> impl axum::response::IntoResponse {
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(Ok(msg)) = socket.next().await {
        if msg.is_text() || msg.is_binary() {
            if socket.send(msg).await.is_err() {
                break; // client disconnected
            }
        }
    }
}

let app = Router::new().route("/ws", any(ws_handler));
```

**Why `.any()`?** The WebSocket HTTP-upgrade mechanism sends an initial `GET` request. Standard HTTP/1.1 clients use GET, but HTTP/2 clients may arrive with a POST due to how hyper (axum's HTTP backend) maps extended CONNECT. Using `.any()` ensures both paths reach your handler. Missing this causes silent 405 rejections on HTTP/2.

---

## 2. WebSocket Message Types (0.8+)

In axum 0.8, message payloads switched from owned `String`/`Vec<u8>` to **zero-copy** types. This avoids allocations when you forward bytes unchanged.

```rust
use axum::extract::ws::Message;
use bytes::Bytes;
use tokio_tungstenite::tungstenite::protocol::CloseFrame;
use axum::extract::ws::Utf8Bytes;

// Text — payload is Utf8Bytes, NOT String
let text_msg = Message::Text(Utf8Bytes::from_static("hello"));

// Extract the inner string when you need &str
if let Message::Text(t) = &text_msg {
    let s: &str = t.as_str();
}

// Binary — payload is Bytes, NOT Vec<u8>
let bin_msg = Message::Binary(Bytes::from_static(b"\x01\x02\x03"));

// Control frames
let ping  = Message::Ping(Bytes::copy_from_slice(b"ping"));
let pong  = Message::Pong(Bytes::copy_from_slice(b"pong"));
let close = Message::Close(Some(CloseFrame {
    code: axum::extract::ws::close_code::NORMAL,
    reason: Utf8Bytes::from_static("bye"),
}));
```

**Why the change?** `Utf8Bytes` wraps `Bytes` internally, so a received text message can be forwarded to another socket without cloning into a `String`. Same for `Binary(Bytes)` over `Vec<u8>`. In a broadcast server this eliminates per-message heap allocations.

---

## 3. WebSocket Configuration

Chain builder methods on `WebSocketUpgrade` _before_ calling `on_upgrade`.

```rust
async fn ws_handler(ws: WebSocketUpgrade) -> impl axum::response::IntoResponse {
    ws.max_frame_size(1024 * 1024)       // 1 MiB per frame
      .max_send_queue(256)               // max queued outgoing messages
      .write_buffer_size(4096)           // per-connection write buffer
      .protocols(["graphql-ws", "v1"])   // advertise sub-protocol list
      .on_upgrade(handle_socket)
}
```

- **max_frame_size** — Rejects frames larger than this. Protects against OOM from malicious clients. Default 16 MiB; lower it for chat apps.
- **max_send_queue** — When the outbound queue is full, `send()` returns an error. Prevents unbounded memory growth when a slow client backs up. Default 100.
- **write_buffer_size** — Internal buffer coalescing small writes into fewer TCP segments.
- **protocols** — Comma-separated list advertised in the `Sec-WebSocket-Protocol` response header. Use `set_selected_protocol` to pick one (see next section).

---

## 4. WebSocket Protocol Selection (0.8.9+)

Clients may request a specific sub-protocol. Inspect `requested_protocols()` and confirm with `set_selected_protocol`.

```rust
use axum::extract::ws::WebSocketUpgrade;

async fn ws_handler(ws: WebSocketUpgrade) -> impl axum::response::IntoResponse {
    let selected = ws
        .requested_protocols()
        .iter()
        .find(|p| *p == "graphql-transport-ws")
        .cloned();

    if let Some(proto) = selected {
        ws.set_selected_protocol(&proto).on_upgrade(handle_socket)
    } else {
        // Reject clients that don't support our protocol
        axum::http::StatusCode::NOT_ACCEPTABLE.into_response()
    }
}
```

**Why it matters?** Sub-protocols let multiplexers (load balancers, API gateways) and client libraries negotiate behavior. Without confirmation the client library typically refuses to open the connection.

---

## 5. WebSocket Patterns

### 5a. Echo Server

```rust
async fn handle_socket(socket: WebSocket) {
    let (mut sender, mut receiver) = socket.split();
    while let Some(Ok(msg)) = receiver.next().await {
        if sender.send(msg).await.is_err() {
            break;
        }
    }
}
```

### 5b. Broadcast Chat Room

```rust
use std::sync::Arc;
use tokio::sync::{broadcast, Mutex};

type SharedState = Arc<Mutex<broadcast::Sender<Message>>>;

#[derive(Clone)]
struct AppState {
    tx: SharedState,
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    axum::extract::State(state): axum::extract::State<AppState>,
) -> impl axum::response::IntoResponse {
    ws.on_upgrade(move |socket| handle_chat(socket, state))
}

async fn handle_chat(socket: WebSocket, state: AppState) {
    let (mut sender, mut receiver) = socket.split();
    let mut rx = state.tx.lock().await.subscribe(); // per-connection receiver

    // Task 1: forward broadcast messages to this client
    let send_task = tokio::spawn(async move {
        while let Ok(msg) = rx.recv().await {
            if sender.send(msg).await.is_err() {
                break;
            }
        }
    });

    // Task 2: receive from this client and broadcast to everyone
    let tx = state.tx.clone();
    let recv_task = tokio::spawn(async move {
        while let Some(Ok(msg)) = receiver.next().await {
            let _ = tx.lock().await.send(msg);
        }
    });

    // When one task ends, abort the other
    tokio::select! {
        _ = send_task => recv_task.abort(),
        _ = recv_task => send_task.abort(),
    }
}

let (tx, _) = broadcast::channel::<Message>(128);
let state = AppState { tx: Arc::new(Mutex::new(tx)) };
let app = Router::new()
    .route("/chat", any(ws_handler))
    .with_state(state);
```

**Why `broadcast`?** It is O(1) per sender (fan-out), unlike `mpsc` which is O(n). Each subscriber gets its own receiver with a configurable lag (messages missed while a slow client catches up are dropped, not buffered).

### 5c. Authentication on Connect

```rust
async fn ws_handler(
    ws: WebSocketUpgrade,
    axum::extract::Query(params): axum::extract::Query<std::collections::HashMap<String, String>>,
) -> impl axum::response::IntoResponse {
    match params.get("token") {
        Some(token) if validate_token(token) => ws.on_upgrade(handle_socket),
        _ => axum::http::StatusCode::UNAUTHORIZED.into_response(),
    }
}

fn validate_token(token: &str) -> bool {
    token == "secret"
}
```

You can also read headers (`axum::http::HeaderMap`) instead of query params. Reject _before_ `on_upgrade` so the TCP connection closes immediately with a 401 — no WebSocket is ever created.

### 5d. Sending and Receiving Concurrently

Always split the socket into sender and receiver. If you `.await` on `recv` inside a loop that also calls `send`, the connection stalls because each direction is sequential on a single task.

```rust
// GOOD — two independent tasks
let (mut sender, mut receiver) = socket.split();
let send_task = tokio::spawn(async move { /* send logic */ });
let recv_task = tokio::spawn(async move { /* recv logic */ });

// BAD — single task blocks on recv, can't send heartbeats
// while let Some(msg) = socket.recv().await { ... }
```

### 5e. Graceful Close Handling

```rust
use axum::extract::ws::Message;

async fn handle_socket(socket: WebSocket) {
    let (mut sender, mut receiver) = socket.split();
    while let Some(result) = receiver.next().await {
        match result {
            Ok(Message::Close(_)) => {
                // Peer initiated close; respond with Close frame (auto by tungstenite)
                break;
            }
            Ok(msg) => {
                if sender.send(msg).await.is_err() {
                    break; // send failure means connection is gone
                }
            }
            Err(e) => {
                eprintln!("WebSocket error: {e}");
                break;
            }
        }
    }
    // tungstenite sends a Close frame automatically on drop
}
```

---

## 6. WebSocket over HTTP/2

HTTP/2 WebSocket uses the extended CONNECT method (RFC 8441). Limitations:

- Not all browsers support it (Chrome 101+, Firefox 100+; Safari requires 17+).
- Intermediate proxies/load balancers must forward CONNECT or the upgrade fails silently.
- Always register with `.any()` as shown in Section 1.

Test by serving with `rustls` (not `openssl`) and connecting from a compatible client.

---

## 7. Server-Sent Events (SSE)

### 7a. Basic SSE Stream

Use `axum::response::sse::{Sse, Event}`. Each `Event` becomes a block of `data:` / `event:` / `id:` / `retry:` lines followed by `\n\n`.

```rust
use axum::response::sse::{Event, Sse, KeepAlive};
use futures_util::stream::{self, Stream};
use std::convert::Infallible;

async fn sse_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = stream::iter(vec![
        Event::default().data("hello"),
        Event::default().data("world").event("greeting"),
        Event::default().data("3").id("msg-3"),
        Event::default().data("retry").retry(3000u64), // ms
        Event::default().comment("heartbeat"),
    ]);

    Sse::new(stream).keep_alive(KeepAlive::default())
}
```

### 7b. Keep-Alive Modes

Without `keep_alive`, if no events are sent the connection may be closed by intermediaries after ~30 s. Choose a strategy:

```rust
// 1. Default — sends a `:` comment every 15 seconds
Sse::new(stream).keep_alive(KeepAlive::default())

// 2. Custom interval (Duration)
use std::time::Duration;
Sse::new(stream).keep_alive(
    KeepAlive::new()
        .interval(Duration::from_secs(10))
        .text("ping"),
)

// 3. Send a real event as heartbeat
Sse::new(stream).keep_alive(
    KeepAlive::new()
        .interval(Duration::from_secs(5))
        .event(Event::default().data("{\"heartbeat\":true}")),
)
```

**Why keep-alive matters?** Browsers auto-reconnect SSE on drop, but many reverse proxies (nginx, Cloudflare) silently kill idle connections. Regular traffic prevents this.

### 7c. JSON Data

```rust
use serde_json::json;

Event::default()
    .event("price-update")
    .json_data(json!({"symbol": "BTC", "price": 67500.42}))
    .unwrap()
```

`.json_data()` serializes to a string and sets the `data:` field. Panics (returns `Err`) if serialization fails — always `.unwrap()` or `.ok()?` in production.

### 7d. Binary Data

For binary payloads (e.g., base64-encoded images in SSE):

```rust
Event::default()
    .binary_data(bytes::Bytes::from_static(b"\x89PNG\r\n"))
    .unwrap()
```

### 7e. Stream Lifetime Management

Return a stream that ends when a watch/tokio-select resolves:

```rust
use tokio_stream::wrappers::BroadcastStream;
use tokio::sync::broadcast;

async fn notifications(tx: broadcast::Sender<String>) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let rx = tx.subscribe();
    let stream = BroadcastStream::new(rx)
        .map(|msg| Ok::<_, Infallible>(Event::default().data(msg.unwrap())));

    Sse::new(stream).keep_alive(KeepAlive::default())
}
```

The SSE connection closes naturally when the stream ends (sender dropped). The client's `EventSource` will auto-reconnect unless you send a special close event.

---

## 8. SSE Patterns

### 8a. Real-Time Notifications

```rust
use tokio::sync::broadcast;
use tokio_stream::wrappers::BroadcastStream;
use tokio_stream::StreamExt;

async fn notifications_handler(
    axum::extract::State(tx): axum::extract::State<broadcast::Sender<String>>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = BroadcastStream::new(tx.subscribe()).map(|msg| {
        Ok(Event::default().event("notification").data(msg.unwrap()))
    });
    Sse::new(stream).keep_alive(KeepAlive::default())
}
```

### 8b. Progress Updates

```rust
async fn progress_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = tokio_stream::iter(0..=100)
        .map(|p| {
            std::thread::sleep(std::time::Duration::from_millis(100));
            Ok::<_, Infallible>(
                Event::default()
                    .event("progress")
                    .data(p.to_string()),
            )
        });
    Sse::new(stream)
}
```

**Production note:** Use `tokio::time::sleep` instead of `std::thread::sleep` to avoid blocking the runtime:

```rust
async fn progress_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = async_stream::stream! {
        for p in 0..=100 {
            tokio::time::sleep(std::time::Duration::from_millis(100)).await;
            yield Ok::<_, Infallible>(
                Event::default().event("progress").data(p.to_string()),
            );
        }
    };
    Sse::new(stream)
}
```

### 8c. Connection Management and Cleanup

Track active connections with `Arc<Mutex<HashSet>>` and clean up on drop:

```rust
use std::collections::HashSet;
use std::sync::{Arc, atomic::{AtomicUsize, Ordering}};

static ACTIVE_CONNECTIONS: AtomicUsize = AtomicUsize::new(0);

async fn sse_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    ACTIVE_CONNECTIONS.fetch_add(1, Ordering::Relaxed);
    let stream = async_stream::stream! {
        for i in 0.. {
            tokio::time::sleep(std::time::Duration::from_secs(2)).await;
            yield Ok::<_, Infallible>(
                Event::default().data(format!("tick {i}"))
            );
        }
    };
    let stream = stream.inspect(|_| {}); // replace with cleanup logic
    Sse::new(stream).keep_alive(KeepAlive::default())
}
```

For per-client cleanup, wrap the stream in a struct with a `Drop` impl, or use `tokio::select!` with a cancellation token.

---

## 9. Choosing Between WebSocket and SSE

| Factor                  | WebSocket                           | SSE                                  |
| ----------------------- | ----------------------------------- | ------------------------------------ |
| **Direction**           | Bidirectional                       | Server → Client only                 |
| **Browser API**         | `new WebSocket(url)`                | `new EventSource(url)`               |
| **Reconnection**        | Manual (write your own)             | Automatic (built-in)                 |
| **Binary data**         | Native frames                       | Base64-encoded in `data:` field      |
| **HTTP/2**              | Extended CONNECT (limited support)  | Fully supported via standard streams |
| **Proxy compatibility** | May need explicit config            | Works through all proxies            |
| **Overhead**            | Framing header (2–14 bytes)         | Text-based, slightly larger          |
| **Use case**            | Chat, gaming, collaborative editing | Notifications, feeds, progress bars  |

**Rule of thumb:** If the client never needs to send real-time data, use SSE — it is simpler, auto-reconnects, and needs no special proxy configuration. Use WebSocket when you need low-latency bidirectional communication.

---

## Common Pitfalls

1. **Using `.get()` instead of `.any()` for WebSocket routes.** HTTP/2 clients send CONNECT which maps to POST in hyper. Without `.any()` they get a 405.

2. **Not splitting the WebSocket.** If you `await` both `recv()` and `send()` on the same task, the connection blocks. Always `.split()` into two tasks.

3. **Assuming `Message::Text` contains `String`.** Since 0.8 it is `Utf8Bytes`. Call `.as_str()` to get `&str`, or `.into_string()` for an owned `String`.

4. **Forgetting `KeepAlive` on SSE streams.** Proxies kill idle connections. The default 15-second comment heartbeat prevents this.

5. **Blocking the tokio runtime inside SSE streams.** Never use `std::thread::sleep` — use `tokio::time::sleep` or `async_stream::stream!` with async awaits.

6. **Unbounded broadcast channels.** `broadcast::channel(usize::MAX)` can OOM. Set a finite capacity (128–1024 is typical) and accept that slow clients will miss messages.

7. **Ignoring send errors.** `sender.send(msg).await.is_err()` means the client disconnected. Always check and break the loop.

8. **SSE does not support custom headers after connection.** You cannot set headers per event. Use the `event:` field to multiplex event types instead.

9. **WebSocket text frames must be valid UTF-8.** Sending invalid UTF-8 in a `Message::Text` panics. Use `Message::Binary` for arbitrary byte payloads.

10. **HTTP/2 WebSocket is not universally supported.** Test with your target browsers. Safari added support in version 17. For maximum compatibility, serve WebSocket over HTTP/1.1 or use SSE as a fallback.
