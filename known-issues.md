# Axum Skill Known Issues

This file documents **real-world** bugs confirmed while testing the skill against the latest crates as of April 2026.

## Environment

- axum 0.8.9
- axum-extra 0.12.6
- axum-macros 0.5.1
- tower-http 0.6.8
- tower 0.5.3
- tokio 1.52.1
- hyper 1.9.0

---

## Issue 1: Legacy path syntax (`:id`, `/*path`) in multiple reference files

**Files affected:**

- `references/routing.md`
- `references/extractors.md`
- `references/testing.md`
- `references/middleware.md`
- `references/state-management.md`

**What the skill says:**
`.route("/users/:id", get(handler))`

**What happens when copied:**

- Compile succeeds but **runtime panic** on startup: `Path segments must not start with :. For capture groups, use {capture}.`
- Catch-all `.route("/*path", ...)` panics with: `Path segments must not start with *. For wildcard capture, use {*wildcard}.`

**Fix:**
Replace all occurrences with new syntax: `:param` → `{param}`, `/*path` → `{*path}`.

---

## Issue 2: `middleware.md` — `next.run()` called with wrong arguments

**What the skill says:**
`next.run(headers).await` and `next.run(()).await` in multiple snippets. The `Next::run` signature takes `Request<Body>`, not `HeaderMap` or `()`.

**What happens when copied:**
Compile error: `expected Request<Body>, found HeaderMap` or `expected Request<Body>, found ()`.

**Fix:**
Change all middleware to take `req: Request<Body>` as the first parameter and call `next.run(req).await`.

---

## Issue 3: `middleware.md` — `next.into_request()` does not exist

**What the skill says:**
`let mut req = next.into_request();` inside authentication middleware example.

**What happens when copied:**
Compile error: `no method named into_request found for struct Next`

**Fix:**
Take `req: Request<Body>` as parameter directly, then do `req.extensions_mut().insert(user); next.run(req).await`.

---

## Issue 4: `references/responses.md` — wrong module path for `Event`

**What the skill says:**
`axum::extract::Event` inside SSE snippet.

**What happens when copied:**
Compile error: `cannot find type Event in module axum::extract`

**Fix:**
`Event` lives in `axum::response::sse`. Replace `axum::extract::Event` with `axum::response::sse::Event`.

---

## Issue 5: `references/tower-http-layers.md` — wrong `CompressionLevel` import

**What the skill says:**
`use tower_http::compression::{CompressionLayer, compression::CompressionLevel};`

**What happens when copied:**
Compile error: `could not find compression in compression`

**Fix:**
`CompressionLevel` is re-exported directly as `tower_http::compression::CompressionLevel` (and also `tower_http::CompressionLevel`).

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

## Test Artifacts

All issues were verified in the `/Users/llama/Developer/test-skills/axum-skills/test-projects/axum-skill-test` Rust project. The project compiles and runs 30+ integration tests using `axum 0.8.9`, confirming each fix.
