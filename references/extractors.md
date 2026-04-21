# Axum 0.8.x Request Extractors — Reference Guide

Extractors pull data out of an incoming HTTP request and hand it to your handler as typed
function parameters. Axum splits them into two families based on whether they read the
request body. Understanding this split is the key to using extractors correctly.

---

## 1. FromRequestParts Extractors (no body consumed)

These implement `FromRequestParts`, meaning they inspect only the request's URI, headers,
and metadata. Multiple parts-extractors can appear as handler parameters in **any order**
because none of them advance the body stream.

### Path\<T\> — URL path segments

Path segments are captured by route definitions and deserialized into the handler type.

```rust
use axum::{routing::get, Router, extract::Path};

// Single captured segment
async fn single(id: Path<u32>) {
    let id: u32 = id.0; // newtype wrapper
}

// Tuple form — matches positional capture groups
async fn tuple(Path((user, repo)): Path<(String, String)>) {
    // route: "/:user/:repo" → Path<(String, String)>
}

// Named struct — order-independent, self-documenting
#[derive(serde::Deserialize)]
struct RepoPath { user: String, repo: String }

async fn named(Path(RepoPath { user, repo }): Path<RepoPath>) {}

let app = Router::new()
    .route("/items/:id", get(single))
    .route("/:user/:repo", get(named));
```

**Why tuples work:** serde can deserialize from a sequence (the captured segments), so
`Path<(A, B)>` works when the route defines two segments.

### Query\<T\> — query string deserialization

```rust
use axum::{extract::Query, response::IntoResponse};

#[derive(serde::Deserialize)]
struct Pagination { page: Option<usize>, per_page: Option<usize> }

async fn list(Query(pag): Query<Pagination>) -> String {
    format!("page={:?}, per_page={:?}", pag.page, pag.per_page)
}

// Parse a query string from any URI without a request (unit test friendly)
fn parse_any_uri() {
    let uri: http::Uri = "/search?q=rust&lang=en".parse().unwrap();
    let Query(params) = Query::<std::collections::HashMap<String, String>>::try_from_uri(&uri).unwrap();
    assert_eq!(params["q"], "rust");
}
```

**Why `try_from_uri` exists:** It constructs the extractor from a bare URI so you can
reuse your deserialization logic in tests and middleware without faking a full request.

### State\<T\> — shared application state

```rust
use axum::{extract::State, routing::get, Router};

#[derive(Clone)]  // Clone is required — every request gets its own copy
struct AppState { db: sqlx::PgPool, version: String }

async fn version(State(state): State<AppState>) -> String {
    state.version.clone()
}

let app = Router::new()
    .route("/version", get(version))
    .with_state(AppState { db: pool, version: "1.0".into() });
```

**Why Clone:** Axum internally wraps state in `Arc`. When a handler calls `State(state)`,
the extractor clones the `Arc` (cheap pointer copy). Your type must be `Clone` so this
mechanism compiles. Use `Arc` inside your state for data you do not want to duplicate.

### Extension\<T\> — runtime-injected per-request data

Middleware adds `Extension` values; handlers read them:

```rust
use axum::extract::Extension;

#[derive(Clone)]
struct RequestId(String);

// Middleware that injects the value
async fn add_request_id(
    mut req: axum::http::Request<axum::body::Body>,
    next: axum::middleware::Next,
) -> axum::response::Response {
    let id = uuid::Uuid::new_v4().to_string();
    req.extensions_mut().insert(RequestId(id));
    next.run(req).await
}

async fn handler(Extension(RequestId(id)): Extension<RequestId>) -> String {
    id
}
```

**Why Extension vs State:** `State` is set once at router construction and is the same for
every request. `Extension` is inserted per-request by middleware, so different requests
can carry different values (auth claims, trace IDs, etc.).

### Other parts extractors

```rust
use axum::extract::{HeaderMap, MatchedPath, OriginalUri, ConnectInfo};
use axum::http::request::Parts;

async fn misc(
    headers: HeaderMap,                    // all headers as HeaderMap
    matched: MatchedPath,                  // e.g. "/items/:id"
    original: OriginalUri,                 // URI before any middleware rewrote it
    ConnectInfo(addr): ConnectInfo<std::net::SocketAddr>, // client IP
    parts: Parts,                          // raw http::request::Parts
) -> String {
    format!("{addr}")
}

// Build the router to populate ConnectInfo (requires `tokio` feature on axum):
// let app = Router::new()
//     .route("/", get(misc))
//     .into_make_service_with_connect_info::<SocketAddr>();
```

**Why `MatchedPath` differs from the actual URI:** A request to `/items/42` has URI
`/items/42`, but `MatchedPath` returns the route pattern `/items/:id`. This is essential
for metrics — you want to group by route, not by every distinct URL.

---

## 2. FromRequest Extractors (consume the body)

These implement `FromRequest`. They **read and exhaust the request body**, so only **one**
can appear per handler and it **must be the last parameter**. If you put a body extractor
before a parts extractor, the code will fail to compile.

### Json\<T\> — JSON request body

```rust
use axum::{extract::Json, routing::post};
use serde::Deserialize;

#[derive(Deserialize)]
struct CreateUser { name: String, email: String }

async fn create_user(Json(payload): Json<CreateUser>) -> &'static str {
    // Json<T> also derefs to T
    &payload.name
}

// Manual JSON parsing without a request (useful in tests):
fn parse_json_bytes() {
    let bytes = b#"{"name":"Alice"}"#;
    let user: CreateUser = serde_json::from_slice(bytes).unwrap();
    // Or equivalently: let user = Json::from_bytes(bytes).unwrap().0;
}
```

**Why content-type is checked:** `Json` rejects requests whose `Content-Type` header is
not `application/json`. This prevents a malformed POST with `text/plain` from silently
producing garbage. Use `axum-extra`'s `JsonDeserializer` for lenient parsing.

### Form\<T\> — URL-encoded body

```rust
use axum::extract::Form;

#[derive(serde::Deserialize)]
struct Login { username: String, password: String }

async fn login(Form(login): Form<Login>) -> &'static str {
    "ok"
}
```

### String and bytes::Bytes — raw body

```rust
async fn raw_text(body: String) -> String {
    format!("got {} bytes", body.len())
}

async fn raw_bytes(body: bytes::Bytes) -> usize {
    body.len()
}
```

**Why `String` may fail:** It requires valid UTF-8. If the body is binary, use `Bytes`.
`Bytes` never fails and is zero-copy (reference-counted).

### Multipart — file uploads (requires `multipart` feature)

```rust
use axum::extract::Multipart;

async fn upload(mut multipart: Multipart) -> &'static str {
    while let Ok(Some(field)) = multipart.next_field().await {
        let name = field.name().unwrap_or_default().to_string();
        if let Ok(data) = field.bytes().await {
            println!("field {name}: {} bytes", data.len());
        }
    }
    "ok"
}

// Cargo.toml: axum = { version = "0.8", features = ["multipart"] }
```

### Request and Body — low-level access

```rust
use axum::{body::Body, http::Request};

async fn full(req: Request<Body>) {}
```

Use these only when you need complete control (e.g., proxying the body elsewhere).

---

## 3. axum-extra Extractors

Add `axum-extra` to your dependencies: `axum-extra = "0.10"`.

### TypedHeader\<T\> — single header, type-safe

```rust
use axum_extra::{TypedHeader, headers::{UserAgent, Authorization, authorization::Bearer}};
use axum_extra::headers;

async fn agent(TypedHeader(ua): TypedHeader<UserAgent>) -> String {
    ua.to_string()
}

async fn bearer(TypedHeader(Auth(bearer)): TypedHeader<Authorization<Bearer>>) -> String {
    bearer.token().to_string()
}
```

**Why TypedHeader exists:** `HeaderMap` gives you raw string values. `TypedHeader<UserAgent>`
parses the header into a typed struct, rejecting malformed values at the extractor level
with a proper 400 response instead of panicking in your handler.

### Host

```rust
use axum_extra::extract::Host;

async fn host(Host(hostname): Host) -> String {
    hostname
}
```

### Cookie jars

```rust
use axum_extra::extract::{CookieJar, Cookie};

async fn set_cookie(jar: CookieJar) -> (CookieJar, &'static str) {
    let jar = jar.add(Cookie::new("session_id", "abc123"));
    (jar, "cookie set")
}

async fn read_cookie(jar: CookieJar) -> Option<&'static str> {
    jar.get("session_id").map(|_| "found")
}
```

`SignedCookieJar` uses HMAC to detect tampering; `PrivateCookieJar` encrypts values.
Both require key configuration at the router level.

### Either\<E1, E2\> — accept multiple body types

```rust
use axum_extra::extract::Either;
use axum::extract::Json;

#[derive(serde::Deserialize)]
struct Payload { data: String }

async fn flexible(
    body: Either<Json<Payload>, String>,
) -> String {
    match body {
        Either::E1(Json(p)) => format!("json: {}", p.data),
        Either::E2(s) => format!("text: {s}"),
    }
}
```

**Why Either:** Without it you would need two separate routes for the same logical
endpoint. `Either` lets a single handler accept JSON or plain text (or any two
FromRequest types) and branch on which one succeeded.

### OptionalQuery\<T\>

```rust
use axum_extra::extract::OptionalQuery;

#[derive(serde::Deserialize)]
struct Filter { tag: Option<String> }

async fn items(OptionalQuery(filter): OptionalQuery<Filter>) -> String {
    match filter {
        Some(f) => format!("filtered by {:?}", f.tag),
        None => "no filter".into(),
    }
}
```

**Why not `Option<Query<T>>`:** In axum 0.8, `Option<Query<T>>` requires the query string
to be _present_ but deserialization may fail. `OptionalQuery<T>` returns `None` when the
entire query string is absent, which is usually what you want.

### WithRejection\<T, R\> — custom rejection types

```rust
use axum::{extract::FromRequest, http::StatusCode, response::{IntoResponse, Response}};

// Replace the default 400 JSON rejection with a plain-text 422
#[derive(Clone)]
struct MyRejection;

impl IntoResponse for MyRejection {
    fn into_response(self) -> Response {
        (StatusCode::UNPROCESSABLE_ENTITY, "invalid payload").into_response()
    }
}

async fn handler(
    body: axum_extra::extract::WithRejection<Json<Payload>, MyRejection>,
) -> &'static str {
    "ok"
}
```

---

## 4. Custom Extractors

Implement `FromRequestParts` for metadata extractors or `FromRequest` for body extractors.
Axum 0.8 uses native `impl Future` — no `#[async_trait]` needed.

### Custom parts extractor (auth token)

```rust
use axum::{
    extract::{FromRequestParts, Request},
    http::{request::Parts, StatusCode, header},
    response::{IntoResponse, Response},
};
use futures::future::BoxFuture;

#[derive(Clone)]
struct AuthToken(String);

struct AuthRejection(StatusCode);

impl IntoResponse for AuthRejection {
    fn into_response(self) -> Response {
        (self.0, "missing or invalid auth token").into_response()
    }
}

// Native async impl — no #[async_trait] in axum 0.8
impl<S> FromRequestParts<S> for AuthToken
where
    S: Send + Sync,
{
    type Rejection = AuthRejection;

    fn from_request_parts(parts: &mut Parts, _state: &S) -> BoxFuture<'_, Result<Self, Self::Rejection>> {
        Box::pin(async move {
            let header = parts
                .headers
                .get(header::AUTHORIZATION)
                .and_then(|v| v.to_str().ok())
                .ok_or(AuthRejection(StatusCode::UNAUTHORIZED))?;

            let token = header
                .strip_prefix("Bearer ")
                .ok_or(AuthRejection(StatusCode::UNAUTHORIZED))?;

            Ok(AuthToken(token.to_string()))
        })
    }
}

async fn protected(AuthToken(token): AuthToken) -> String {
    token
}
```

### Custom body extractor

Implement `FromRequest` the same way, but receive `Request<Body>` instead of `&mut Parts`.
Call `request.into_body()` to obtain the body stream.

---

## 5. Option\<T\> and Result\<T, E\> Patterns

In axum 0.8, `Option<T>` wrapping an extractor uses the `OptionalFromRequestParts` /
`OptionalFromRequest` traits internally:

- **`Option<Query<T>>`** — succeeds even if the query string is present but empty or
  malformed; returns `None` on _any_ failure. This means a typo like `?pag=1` (instead of
  `?page=1`) silently gives `None` rather than a 400 error.
- **`Option<TypedHeader<UserAgent>>`** — returns `None` if the header is missing or
  malformed.
- **`Result<Query<T>, QueryRejection>`** — returns the rejection so you can produce a
  custom error response.

```rust
async fn flexible(
    // Will not reject if query is absent
    opt: Option<Query<Pagination>>,
) -> String {
    match opt {
        Some(Query(p)) => format!("page {:?}", p.page),
        None => "no params".into(),
    }
}
```

**Key change from 0.7:** In 0.7, `Option<T>` used a different internal mechanism that
could silently swallow parse errors. In 0.8, `OptionalFromRequestParts` is the dedicated
trait, and its behavior is more predictable — it always returns `None` on failure.

---

## 6. Extractor Ordering Rules

Axum enforces ordering at **compile time** through Rust's type system. The framework
generates a tuple implementation of `FromRequestParts` / `FromRequest` for your handler
parameters. The rules:

1. **All `FromRequestParts` extractors** can appear in any order and in any quantity.
2. **At most one `FromRequest` extractor** is allowed per handler.
3. **The `FromRequest` extractor must be the last parameter** (or the only parameter).

```rust
// ✅ Compiles — parts, parts, body (last)
async fn ok(
    Query(params): Query<Params>,
    State(state): State<AppState>,
    Json(body): Json<Payload>,
) -> Response { todo!() }

// ❌ Compile error — body extractor not last
// async fn bad(
//     Json(body): Json<Payload>,
//     Query(params): Query<Params>,  // compiler rejects this
// ) {}
```

**Why this is a compile-time error:** Axum's handler macro generates an extractor chain.
The tuple `(Query<T>, Json<U>)` tries to impl `FromRequest`, which calls `FromRequestParts`
for each element and `FromRequest` for the last. If the `FromRequest` element is not last,
the generated code cannot type-check because body has already been moved.

---

## Common Pitfalls

1. **Forgetting `Clone` on State** — The state type must implement `Clone`. Wrap expensive
   fields in `Arc` to avoid deep copies.

2. **`Option<Query<T>>` silently swallowing errors** — A malformed query string (e.g.
   `?page=abc` for a `usize` field) returns `None` instead of a 400. Use `Query<T>`
   directly if you want validation errors, or `OptionalQuery<T>` from axum-extra if you
   only want to treat _absence_ as optional.

3. **Multiple body extractors** — You cannot have both `Json<T>` and `Form<U>` in the same
   handler. The body stream can only be read once. Use `Either` from axum-extra if you need
   to accept different content types on the same route.

4. **Using `String` body for binary data** — `String` requires valid UTF-8 and will reject
   binary payloads with a 400. Use `bytes::Bytes` instead.

5. **Missing `multipart` feature** — The `Multipart` extractor requires the `multipart`
   cargo feature on axum. Without it, the type does not exist.

6. **ConnectInfo not populated** — `ConnectInfo<SocketAddr>` only works when the server is
   created with `into_make_service_with_connect_info::<SocketAddr>()`, not with the default
   `into_make_service()`.

7. **Extension type collisions** — Inserting two values of the same type into `Extension`
   silently overwrites the first. Use a newtype wrapper (e.g., `struct AuthUser(User)`) to
   avoid collisions.
