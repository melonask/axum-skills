# axum — Rust Web Framework Skill

A comprehensive skill for building web applications and APIs with [axum](https://github.com/tokio-rs/axum) (by tokio-rs). This skill enables LLM developers to build production-ready HTTP services in Rust using axum's ergonomic handler-based API, its deep integration with the Tower middleware ecosystem, and the full power of async Rust via Tokio.

```bash
npx skills add melonask/axum-skills
```

## What This Skill Covers

This skill provides complete guidance for every major feature of the axum framework (version 0.8.x), including:

- **Routing** — Path parameters, nested routers, wildcard routes, fallbacks, method routing, route merging
- **Extractors** — Path, Query, Json, Form, Bytes, String, State, Extension, HeaderMap, WebSocket, Multipart, cookies, TypedHeader, Either/N types, and custom extractors
- **Responses** — IntoResponse trait, IntoResponseParts, status codes, headers, HTML, JSON, streaming, redirects
- **Middleware** — `middleware::from_fn`, `from_fn_with_state`, `map_request`, `map_response`, tower layers, per-route middleware
- **State Management** — Shared application state, FromRef, Extension, Arc patterns
- **Error Handling** — Custom error types, rejection types, WithRejection, BoxError
- **Real-Time** — WebSocket (HTTP/1 and HTTP/2), Server-Sent Events (SSE), broadcast patterns
- **File Handling** — Multipart uploads, static file serving, SPA fallback
- **Cookies** — Plain, signed (HMAC), and private (AES-encrypted) cookie management
- **Production Layers** — CORS, tracing, compression, timeouts, rate limiting, auth, request IDs via tower-http
- **Testing** — Unit and integration testing with tower::ServiceExt
- **Migration** — Upgrading from axum 0.7 to 0.8

## Skill Structure

```
axum/
├── SKILL.md                          # Core skill instructions (entry point)
├── README.md                         # This file — skill overview and installation
└── references/                       # Deep-dive guides loaded on demand
    ├── routing.md                    # Route definition, path params, nesting, fallbacks
    ├── extractors.md                 # All extractors (axum + axum-extra)
    ├── responses.md                  # IntoResponse, IntoResponseParts, response builders
    ├── middleware.md                  # from_fn, from_fn_with_state, map_request/response
    ├── state-management.md           # State, FromRef, Extension, Arc patterns
    ├── error-handling.md             # Custom errors, rejections, WithRejection
    ├── realtime.md                   # WebSocket, SSE, broadcast patterns
    ├── files-uploads.md              # Multipart uploads, static file serving
    ├── cookies.md                    # CookieJar, SignedCookieJar, PrivateCookieJar
    ├── tower-http-layers.md          # All tower-http middleware layers
    ├── testing.md                    # Unit and integration testing patterns
    └── migration-0.8.md              # Migration guide from axum 0.7 to 0.8
```

## How It Works

The skill uses a **progressive disclosure** architecture:

1. **SKILL.md** (always loaded) — Provides crate architecture, quick-start setup, feature flags, 6 core code patterns, key concepts, and a routing table pointing to reference files
2. **references/\*.md** (loaded on demand) — When a user's request involves a specific area (e.g., middleware, cookies, WebSocket), the LLM reads the corresponding reference file for in-depth guidance with additional code examples and edge cases

This keeps the initial context small while making detailed information available when needed.

## Compatible Versions

| Crate         | Version                 |
| ------------- | ----------------------- |
| `axum`        | 0.8.x (latest: 0.8.9)   |
| `axum-core`   | 0.5.x (latest: 0.5.6)   |
| `axum-extra`  | 0.12.x (latest: 0.12.6) |
| `axum-macros` | 0.5.x (latest: 0.5.1)   |
| `tower-http`  | 0.6.x                   |
| `tower`       | 0.5.x                   |
| `tokio`       | 1.x                     |
| `hyper`       | 1.4+                    |

## Quick Start

```toml
# Cargo.toml
[dependencies]
axum = "0.8.9"
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use axum::{Router, routing::get, serve};

let app = Router::new().route("/", get(|| async { "Hello, World!" }));

let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
serve(listener, app).await.unwrap();
```

## Key Design Principles

- **Handler-first**: Axum converts ordinary async functions into HTTP handlers via trait implementations (`FromRequest`, `IntoResponse`)
- **Tower-native**: Every axum `Router` is a `tower::Service`, so any tower middleware works without adapters
- **Type-safe**: Path parameters, query strings, and request bodies are deserialized into typed structs via serde
- **Ergonomic**: Extractors as function parameters, tuple responses for status + body, automatic error propagation
- **Zero-cost abstractions**: No runtime overhead beyond what Tokio and Hyper already provide

## License

This skill is provided as-is for educational and development purposes. axum itself is licensed under the MIT License.
