# Axum 0.8 Response Types Reference

Every axum handler returns a type that implements `IntoResponse`. This trait is the single
bridge between your handler's return value and the actual HTTP response sent to the client.

---

## 1. The `IntoResponse` Trait

`IntoResponse` converts any Rust value into an `http::Response<axum::body::Body>`. Axum calls
`.into_response()` on your handler's return value before sending it over the wire. You rarely
call this method yourself — axum does it internally — but understanding it is the key to
understanding every other topic in this file.

```rust
use axum::{body::Body, http::Response, response::IntoResponse};

// Any type implementing IntoResponse can produce a full HTTP response.
fn inspect<T: IntoResponse>(value: T) -> Response<Body> {
    value.into_response()
}
```

---

## 2. Built-in Implementations

### Plain Text

`&'static str`, `String`, `Box<str>`, and `Cow<'static, str>` all produce a `200 OK` with
`Content-Type: text/plain; charset=utf-8`.

```rust
use axum::routing::get;
use axum::Router;

async fn plain_static() -> &'static str { "hello" }
async fn plain_owned() -> String { "hello".to_owned() }
async fn plain_cow() -> std::borrow::Cow<'static, str> {
    std::borrow::Cow::Borrowed("hello")
}

let app = Router::new()
    .route("/static", get(plain_static))
    .route("/owned", get(plain_owned))
    .route("/cow", get(plain_cow));
```

### Binary

`&'static [u8]`, `Vec<u8>`, `Box<[u8]>`, `Cow<'static, [u8]>`, `Bytes`, and `BytesMut` all
produce `200 OK` with `Content-Type: application/octet-stream`.

```rust
use axum::routing::get;
use bytes::{Bytes, BytesMut};

async fn binary_slice() -> &'static [u8] { b"\x89PNG\r\n" }
async fn binary_vec() -> Vec<u8> { vec![0x89, 0x50, 0x4e, 0x47] }
async fn binary_bytes() -> Bytes { Bytes::from_static(b"\x89PNG\r\n") }
async fn binary_bytes_mut() -> BytesMut { BytesMut::from(&b"\x89PNG\r\n"[..]) }

let app = Router::new()
    .route("/slice", get(binary_slice))
    .route("/vec", get(binary_vec))
    .route("/bytes", get(binary_bytes))
    .route("/bytes-mut", get(binary_bytes_mut));
```

### StatusCode and Unit

`StatusCode` sends the status code with an empty body. `()` is equivalent to `StatusCode::OK`
with an empty body. `Infallible` can never be constructed, so it signals a handler that
never fails (useful in `Result<T, Infallible>`).

```rust
use axum::http::StatusCode;

async fn created() -> StatusCode { StatusCode::CREATED }
async fn no_body() -> () { () }
async fn infallible() -> std::convert::Infallible {
    unreachable!("this branch is never taken")
}
```

### `Html<T>`

Wraps any `T: IntoResponse` and sets `Content-Type: text/html; charset=utf-8`. The inner
value's original content-type is overwritten.

```rust
use axum::response::Html;

async fn index() -> Html<&'static str> {
    Html("<h1>Hello</h1>")
}

// Html also works with owned strings or templates.
async fn rendered() -> Html<String> {
    Html(format!("<h1>Hello, {}!</h1>", "world"))
}
```

### `Json<T>`

Serializes `T` as JSON with `Content-Type: application/json`. Requires `T: Serialize` and
the `json` feature flag on axum (enabled by default). On serialization failure the client
receives `500 Internal Server Error`.

```rust
use axum::Json;
use serde_json::{json, Value};

#[derive(serde::Serialize)]
struct User { id: u64, name: String }

async fn user_json() -> Json<User> {
    Json(User { id: 1, name: "Alice".into() })
}

async fn ad_hoc() -> Json<Value> {
    Json(json!({ "status": "ok" }))
}
```

### Raw Body and Full Response

`axum::body::Body` passes through untouched. `http::Response<Body>` is already a complete
response — axum returns it as-is without modification.

```rust
use axum::{body::Body, http::{Response, header}};
use bytes::Bytes;

async fn raw_body() -> Body {
    Body::from("raw bytes")
}

async fn full_response() -> Response<Body> {
    Response::builder()
        .status(201)
        .header(header::CONTENT_TYPE, "application/json")
        .body(Body::from(r#"{"created":true}"#))
        .unwrap()
}
```

### `Result<T, E>`

When both `T` and `E` implement `IntoResponse`, `Result<T, E>` does too. On `Ok(value)` the
happy-path response is returned; on `Err(e)` the error response is returned. This is the
standard pattern for fallible handlers.

```rust
use axum::{http::StatusCode, Json};
use serde_json::json;

async fn fallible() -> Result<Json<serde_json::Value>, StatusCode> {
    if true {
        Ok(json!({ "data": 42 }))
    } else {
        Err(StatusCode::NOT_FOUND)
    }
}

// With a custom error type (see section 5 for the impl):
// async fn with_custom_err() -> Result<Json<User>, AppError> { ... }
```

### `Sse<Stream>` — Server-Sent Events

`Sse` wraps any stream of `Event` items and sets `Content-Type: text/event-stream`. The
connection stays open; each yielded event is sent immediately. Requires the `macros` feature
for the `axum::extract::Event` helpers.

```rust
use axum::{response::Sse, routing::get, Router};
use futures::stream::{self, Stream};
use std::convert::Infallible;
use tokio_stream::StreamExt;

async fn sse_handler() -> Sse<impl Stream<Item = Result<axum::extract::Event, Infallible>>> {
    let stream = stream::iter(vec![
        Ok(axum::extract::Event::default().data("hello")),
        Ok(axum::extract::Event::default().data("world")),
    ]);
    Sse::new(stream)
}

let app = Router::new().route("/sse", get(sse_handler));
```

### `WebSocketUpgrade` — WebSocket Upgrade

Returns a 101 Switching Protocols response after the client confirms the upgrade. Call
`.on_upgrade(callback)` to provide the async function that handles the socket.

```rust
use axum::{
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    routing::get,
    Router,
};

async fn ws_upgrade(ws: WebSocketUpgrade) -> impl axum::response::IntoResponse {
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(msg) = socket.recv().await {
        if let Ok(Message::Text(text)) = msg {
            let _ = socket.send(Message::Text(text)).await;
        }
    }
}

let app = Router::new().route("/ws", get(ws_upgrade));
```

### `Redirect`

Produces a 3xx response with a `Location` header. Several constructors cover the common
cases.

```rust
use axum::response::Redirect;

async fn temp_redirect() -> Redirect {
    Redirect::temporary("https://example.com")
}
async fn permanent_redirect() -> Redirect {
    Redirect::permanent("https://example.com")
}
async fn see_other() -> Redirect {
    Redirect::with_status_code(
        "https://example.com/after-post",
        axum::http::StatusCode::SEE_OTHER,
    )
}
```

### `NoContent` (axum 0.8)

Returns `204 No Content` with an empty body. Use it instead of `StatusCode::NO_CONTENT` when
you want a named, self-documenting type.

```rust
use axum::response::NoContent;

async fn delete_item() -> NoContent {
    // Perform deletion...
    NoContent
}
```

---

## 3. Tuple Responses

Tuples let you combine a status code (and optionally headers) with a body. Each element
must implement either `IntoResponse` or `IntoResponseParts`. The **last** element becomes the
body; earlier elements modify the response's status and headers.

### Status + Body

```rust
use axum::{http::StatusCode, Json};

async fn created_user() -> (StatusCode, Json<serde_json::Value>) {
    (StatusCode::CREATED, Json(json!({ "id": 1 })))
}
```

### Status + Array Headers + Body

An array of `(K, V)` pairs adds headers. Each key type must implement `HeaderName`-like
conversion.

```rust
use axum::{http::{StatusCode, header}, Json};

async fn with_headers() -> (StatusCode, [(header::HeaderName, &str); 2], Json<&str>) {
    (
        StatusCode::ACCEPTED,
        [
            (header::X_REQUEST_ID, "abc-123"),
            (header::CACHE_CONTROL, "no-cache"),
        ],
        Json("done"),
    )
}
```

### Status + HeaderMap + Body

```rust
use axum::{http::{StatusCode, HeaderMap, header}, Json};

async fn header_map_response() -> (StatusCode, HeaderMap, Json<&str>) {
    let mut headers = HeaderMap::new();
    headers.insert(header::X_REQUEST_ID, "xyz".parse().unwrap());
    (StatusCode::OK, headers, Json("with header map"))
}
```

### Parts + Body

`http::response::Parts` gives full control over status, headers, version, and extensions.

```rust
use axum::{body::Body, http::{Response, StatusCode, header}};

async fn parts_response() -> (http::response::Parts, Body) {
    let mut resp = Response::new(Body::from("custom parts"));
    let (parts, body) = resp.into_parts();
    // Modify parts before returning...
    (parts, body)
}
```

---

## 4. `IntoResponseParts` — Modifying Response Metadata

`IntoResponseParts` lets a type contribute headers or extensions to a response **without**
owning the body. It is what makes the tuple syntax work: the non-final tuple elements
implement `IntoResponseParts`, not `IntoResponse`.

### `HeaderMap` and Header Arrays

```rust
use axum::{http::HeaderMap, response::IntoResponseParts};

// Returns only parts — useful as a non-final tuple element.
fn custom_headers() -> HeaderMap {
    let mut map = HeaderMap::new();
    map.insert("x-custom", "value".parse().unwrap());
    map
}

// Array syntax also works:
fn header_array() -> [(&str, &str); 1] {
    [("x-foo", "bar")]
}
```

### `Extensions`

Inject arbitrary typed data into the response's extension map. Middleware or test harnesses
can read it back later.

```rust
use axum::http::Extensions;

fn inject_extension() -> Extensions {
    let mut ext = Extensions::new();
    ext.insert("request_id");
    ext
}
```

### `AppendHeaders`

`AppendHeaders` is a stable way to attach multiple headers to a response. It implements
`IntoResponseParts` directly.

```rust
use axum::response::AppendHeaders;
use axum::http::header;

fn multi_headers() -> AppendHeaders<[(header::HeaderName, &str); 2]> {
    AppendHeaders([
        (header::CACHE_CONTROL, "max-age=3600"),
        (header::VARY, "Accept-Encoding"),
    ])
}
```

---

## 5. Custom `IntoResponse` Implementations

Implementing `IntoResponse` on your own types gives you full control. A common pattern is an
enum whose variants map to different HTTP statuses with JSON error bodies.

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;

#[derive(Debug)]
pub enum AppError {
    NotFound(String),
    Unauthorized,
    InternalServerError(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::NotFound(msg) => (StatusCode::NOT_FOUND, msg),
            AppError::Unauthorized => (
                StatusCode::UNAUTHORIZED,
                "authentication required".into(),
            ),
            AppError::InternalServerError(msg) => {
                (StatusCode::INTERNAL_SERVER_ERROR, msg)
            }
        };
        (status, Json(json!({ "error": message }))).into_response()
    }
}

// Now use it as the error branch of Result:
async fn get_user() -> Result<Json<serde_json::Value>, AppError> {
    Err(AppError::NotFound("user not found".into()))
}
```

You can also implement it for structs when a single type always produces the same shape:

```rust
pub struct ApiOk<T>(pub T);

impl<T: serde::Serialize> IntoResponse for ApiOk<T> {
    fn into_response(self) -> Response {
        (StatusCode::OK, Json(serde_json::json!({ "data": self.0 }))).into_response()
    }
}
```

---

## 6. Response Builder Patterns

When you need fine-grained control, build the response manually with `Response::builder()`.

```rust
use axum::{body::Body, http::{Response, header, StatusCode}};

async fn hand_built() -> Response<Body> {
    Response::builder()
        .status(StatusCode::CREATED)
        .header(header::CONTENT_TYPE, "application/json")
        .header(header::LOCATION, "/items/42")
        .body(Body::from(r#"{"id":42}"#))
        .expect("valid status and headers")
}
```

`Response::builder()` returns an `http::response::Builder`. Calling `.body()` consumes the
builder and produces `Result<Response<Body>, http::Error>`. The error only occurs when the
status code is invalid or a header name/value is malformed — it cannot fail at runtime for
well-known constants.

---

## 7. Streaming Responses

For large payloads or real-time data, stream the body instead of buffering it entirely.

### `Body::from_stream`

Convert any `TryStream<Ok = Bytes, Error = E>` into a response body. Axum automatically uses
chunked transfer encoding.

```rust
use axum::{body::Body, routing::get, Router};
use futures::stream;
use tokio_stream::StreamExt;

async fn stream_bytes() -> Body {
    let chunks: Vec<Result<bytes::Bytes, std::convert::Infallible>> = vec![
        Ok(bytes::Bytes::from("chunk 1\n")),
        Ok(bytes::Bytes::from("chunk 2\n")),
        Ok(bytes::Bytes::from("chunk 3\n")),
    ];
    Body::from_stream(stream::iter(chunks))
}

let app = Router::new().route("/stream", get(stream_bytes));
```

### Chunked Transfer with `StreamBody`

`StreamBody` (from `axum::body::StreamBody`) wraps a stream and implements `IntoResponse`
directly. Content-Type must be set separately via a tuple.

```rust
use axum::{
    body::StreamBody,
    http::{header, StatusCode},
    response::IntoResponse,
    routing::get,
    Router,
};
use futures::stream;
use tokio_stream::StreamExt;

async fn chunked_csv() -> impl IntoResponse {
    let stream = stream::iter(vec![
        Ok::<_, std::convert::Infallible>(bytes::Bytes::from("id,name\n")),
        Ok(bytes::Bytes::from("1,Alice\n")),
        Ok(bytes::Bytes::from("2,Bob\n")),
    ]);
    (
        StatusCode::OK,
        [(header::CONTENT_TYPE, "text/csv")],
        StreamBody::new(stream),
    )
}

let app = Router::new().route("/csv", get(chunked_csv));
```

### SSE (revisited with a real stream)

SSE is a special case of streaming where each event is framed with `data:`, `event:`, and
`id:` fields.

```rust
use axum::{response::Sse, routing::get, Router, extract::Event};
use futures::stream::{self, Stream};
use std::{time::Duration, convert::Infallible};
use tokio_stream::StreamExt;

async fn ticking_clock() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = tokio_stream::wrappers::IntervalStream::new(tokio::time::interval(
        Duration::from_secs(1),
    ))
    .map(|_| {
        Ok(Event::default().data(
            chrono::Utc::now().to_rfc3339(),
        ))
    });
    Sse::new(stream).keep_alive(
        axum::response::sse::KeepAlive::new()
            .interval(Duration::from_secs(15))
            .text("ping"),
    )
}
```

---

## 8. Content-Type Handling

Axum infers `Content-Type` from the IntoResponse implementation you use:

| Type                                     | Content-Type                |
| ---------------------------------------- | --------------------------- |
| `&str`, `String`, `Box<str>`, `Cow<str>` | `text/plain; charset=utf-8` |
| `&[u8]`, `Vec<u8>`, `Bytes`              | `application/octet-stream`  |
| `Html<T>`                                | `text/html; charset=utf-8`  |
| `Json<T>`                                | `application/json`          |
| `Sse<Stream>`                            | `text/event-stream`         |
| `Body`, `StreamBody`                     | **none set** (set manually) |
| `StatusCode`, `()`, `NoContent`          | **none** (no body)          |

When you use a tuple like `(StatusCode, Json<T>)`, the `Json<T>` content-type wins. When
you use `Body` or `StreamBody` directly, no content-type is set automatically — always
explicitly add one via a header in a tuple.

```rust
use axum::{
    body::Body,
    http::{header, StatusCode},
    response::IntoResponse,
};

// WRONG: Body with no content-type.
// async fn bad() -> Body { Body::from("<h1>hi</h1>") }

// CORRECT: Set content-type explicitly.
async fn html_body() -> impl IntoResponse {
    (
        [(header::CONTENT_TYPE, "text/html; charset=utf-8")],
        Body::from("<h1>hi</h1>"),
    )
}
```

---

## Common Pitfalls

**Returning a reference to a local.** `&'static str` requires the string to live for the
entire program. Returning `&String` or `&str` borrowed from a local variable will not
compile because the lifetime is too short.

```rust
// This will NOT compile:
// async fn bad() -> &'static str {
//     let s = String::from("hello");
//     &s  // s is dropped here
// }

// Fix: return an owned type.
async fn good() -> String { String::from("hello") }
```

**Missing Content-Type on raw bodies.** `Body` and `StreamBody` do not set a Content-Type.
Clients (especially browsers) may misinterpret the response. Always pair them with a header.

**Forgetting to `.await` in handlers.** If your handler returns `impl Future`, the compiler
error about `IntoResponse` not being implemented often masks the real issue. Ensure every
`.await` is present.

**Result with non-IntoResponse error.** `Result<T, E>` only implements `IntoResponse` when
both `T: IntoResponse` and `E: IntoResponse`. If `E` is `std::io::Error`, it will not
compile. Map it into a type that implements `IntoResponse` (e.g., `StatusCode` or a custom
enum) using `.map_err()`.

**Chunked vs Content-Length.** Axum uses chunked transfer encoding for streaming bodies
automatically. If you need Content-Length instead, buffer the full response first and set the
header manually.

**Json serialization panics.** `Json<T>` uses `serde_json::to_string` internally. If
serialization fails, the client receives a 500 with no body. Validate data before wrapping
in `Json`, or use `serde_json::to_value` + `Json` on the result to control the error path.
