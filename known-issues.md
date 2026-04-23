# Axum Skill Known Issues

This file documents **real-world** bugs confirmed while testing the skill against the latest crates as of April 2026.

## Environment

- axum 0.8.9
- axum-core 0.5.6
- axum-extra 0.12.6
- axum-macros 0.5.1
- tower-http 0.6.8
- tower 0.5.3
- tokio 1.52.1
- hyper 1.9.0
- cookie 0.18.1
- matchit 0.8.6

---

## Issue 1: Legacy path syntax (`:id`, `/*path`) in multiple reference files

**Files affected:**

- `references/routing.md`
- `references/extractors.md`
- `references/testing.md`
- `references/middleware.md`
- `references/state-management.md`
- `references/migration-0.8.md`

**What the skill says:**
`.route("/users/:id", get(handler))`

**What happens when copied:**

- Compile succeeds but **runtime panic** on startup: `Path segments must not start with :. For capture groups, use {capture}.`
- Catch-all `.route("/*path", ...)` panics with: `Path segments must not start with *. For wildcard capture, use {*wildcard}.`

**Fix:**
Replace all occurrences with new syntax: `:param` → `{param}`, `/*path` → `{*path}`.

---

## Issue 2: `middleware.md` — `next.run()` called with wrong arguments

**Files affected:**

- `references/middleware.md`
- `references/testing.md`
- `references/state-management.md`

**What the skill says:**
`next.run(headers).await`, `next.run(()).await`, and `next.run(()).await` in multiple snippets.

**What happens when copied:**
Compile error: `expected Request<Body>, found HeaderMap` or `expected Request<Body>, found ()`.

**Fix:**
Change all middleware to take `req: Request<Body>` (or `req: Request`) as the first parameter and call `next.run(req).await`.

---

## Issue 3: `middleware.md` — `next.into_request()` does not exist

**What the skill says:**
`let mut req = next.into_request();` inside the custom middleware with extractors example.

**What happens when copied:**
Compile error: `no method named into_request found for struct Next`

**Fix:**
Take `req: Request<Body>` as parameter directly, then do `req.extensions_mut().insert(user); next.run(req).await`.

---

## Issue 4: `references/responses.md` — wrong module path for `Event`

**What the skill says:**
`axum::extract::Event` or `axum::response::Event` inside SSE snippets.

**What happens when copied:**
Compile error: `cannot find type Event in module axum::extract`

**Fix:**
`Event` lives in `axum::response::sse`. Replace with `axum::response::sse::Event`.

---

## Issue 5: `references/tower-http-layers.md` — wrong `CompressionLevel` import

**What the skill says:**
`use tower_http::compression::{CompressionLayer, compression::CompressionLevel};`

**What happens when copied:**
Compile error: `could not find compression in compression`

**Fix:**
`CompressionLevel` is re-exported directly as `tower_http::compression::CompressionLevel`.

---

## Issue 6: `references/tower-http-layers.md` — `RequireAuthorizationLayer` does not exist

**What the skill says:**
Uses `tower_http::auth::RequireAuthorizationLayer::basic` and `::bearer`.

**What happens when copied:**
Compile error: `could not find RequireAuthorizationLayer in auth`

**Fix:**
In tower-http 0.6, only `AsyncRequireAuthorizationLayer` exists. Provide an example using `AsyncAuthorizeRequest` trait + `AsyncRequireAuthorizationLayer::new(...)`.

---

## Issue 7: `references/cookies.md` — outdated `axum-extra` version

**What the skill says:**
`axum-extra = { version = "0.10", features = [...] }`

**Impact:**
Mismatched with SKILL.md which correctly recommends `0.12.6`. Copying this Cargo.toml snippet alone would downgrade dependencies and possibly introduce compatibility issues.

**Fix:**
Change version to `0.12.6`.

---

## Issue 8: `SKILL.md` — SSE snippet uses `futures_core::stream::Stream` without dependency note

**What the skill says:**
`impl futures_core::stream::Stream<Item = Result<Event, Infallible>>`

**Impact:**
`futures_core` is not in the skill's quick-start `Cargo.toml` snippet. Most users only have `futures-util`. Code won't compile without adding `futures-core = "0.3"`.

**Fix:**
Replace with `futures_util::stream::Stream` (same trait, re-exported) or add `futures-core` to the snippet.

---

## Issue 9: `migration-0.8.md` — `Host` feature version also outdated

**What the skill says:**
`axum-extra = { version = "0.10", features = ["host"] }`.

**Fix:**
Change to `0.12` (or `0.12.6`).

---

## Issue 10: `extractors.md` custom extractor uses `BoxFuture` needlessly

**What the skill says:**
Custom extractor example still uses `BoxFuture` and `Box::pin` for `from_request_parts`.

**Impact:**
While the code compiles, it requires `futures` crate dependency. In axum 0.8, `from_request_parts` natively returns `impl Future`, so `async fn` works directly without `BoxFuture`.

**Fix:**
Rewrite example with `async fn from_request_parts(...)`.

---

## Issue 11: `references/cookies.md` — `cookie::Duration` is private

**What the skill says:**
`use cookie::{Cookie, SameSite, Duration, time::OffsetDateTime};`

**What happens when copied:**
Compile error: `struct Duration is private`

**Fix:**
`Duration` is re-exported from the `time` crate. Use `use cookie::time::Duration;` instead.

**Also:**
`CookieBuilder::finish()` is deprecated in `cookie` 0.18+; prefer `CookieBuilder::build()`.

---

## Issue 12: `references/tower-http-layers.md` — `TimeoutLayer::new` is deprecated

**What the skill says:**
`.layer(TimeoutLayer::new(Duration::from_secs(3)))`

**What happens when copied:**
Deprecation warning: `use of deprecated associated function tower_http::timeout::TimeoutLayer::new`

**Fix:**
Use `TimeoutLayer::with_status_code`.

---

## Issue 13: `references/tower-http-layers.md` — `SetHeaderLayer` module path changed

**What the skill says:**
`use tower_http::{set_header::SetHeaderLayer, set_status::SetStatusLayer};`

**What happens when copied:**
Compile error for `SetHeaderLayer`.

**Fix:**
`SetHeaderLayer` lives in `tower_http::set_header`. Verify `set-header` feature is enabled.

---

## Issue 14: `references/files-uploads.md` — `MultipartError` does not exist in `axum_extra::extract`

**What the skill says:**
`use axum_extra::extract::MultipartError;`

**What happens when copied:**
Compile error: `no MultipartError in extract`

**Fix:**
Remove this import; use `axum::extract::multipart::MultipartError` or handle errors via `Result`.

---

## Issue 15: `references/files-uploads.md` — `field.file_name()` returns `Option<&str>`, not `Option<String>`

**What the skill says:**
`let _name: Option<String> = field.file_name();`

**What happens when copied:**
Compile error: `mismatched types: expected Option<String>, found Option<&str>`

**Fix:**
`field.file_name()` returns `Option<&str>`. Call `.map(|s| s.to_string())` if you need an owned `String`.

---

## Issue 16: `references/testing.md` — `StatusCode::NOT_FOUND.into_response()` doesn't compile without import

**What the skill says:**
`axum::http::StatusCode::NOT_FOUND.into_response()` inside handler.

**What happens when copied:**
Compile error: `no method named into_response found for struct StatusCode`

**Fix:**
Import `axum::response::IntoResponse`.

---

## Issue 17: `references/migration-0.8.md` — old `*` catch-all syntax causes panic

**What the skill says:**
`.route("/files/*path", get(serve_file))` in the "Before" section.

**What happens when copied:**
The snippet is intentionally "Before", but if a user copies it, they get a runtime panic.

**Fix:**
Already labeled "Before", but add a clearer warning that this syntax will panic in 0.8.

---

## Issue 18: `references/middleware.md` — `HeaderMap` extractor in middleware without `Request` parameter

**What the skill says:**
```rust
async fn auth_middleware(
    headers: axum::http::HeaderMap,
    next: middleware::Next,
) -> Result<impl IntoResponse, AuthError> {
```

**What happens when copied:**
Compile error because `HeaderMap` is a `FromRequestParts` extractor that can't be used alone in `from_fn` without `Request`.

**Fix:**
Change to take `req: Request<Body>` and read `req.headers()` inside the function.

---

## Issue 19: `axum_extra::extract::Host` is deprecated in axum-extra 0.12.6

**What the skill says:**
Using `Host` extractor for hostname extraction.

**What happens when copied:**
Deprecation warning:
```
use of deprecated tuple struct `axum_extra::extract::Host`: will be removed in the next version
```

**Fix:**
Use `http::HeaderMap` and manually read the `Host` header, or watch for the next axum-extra release for the replacement.

---

## Test Artifacts

All issues were verified in the `/tmp/axum-skill-test` Rust project against real crates as of April 2026. The project runs 6+ unit tests and a full integration test suite against a real axum server covering routing, extractors, error handling, WebSocket, SSE, cookies, form handling, file downloads, authentication middleware, and custom response headers.
