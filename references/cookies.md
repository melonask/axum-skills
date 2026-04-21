# Axum Cookie Management Reference (axum 0.8.x)

## Dependency Setup

Add `axum-extra` with the appropriate cookie feature flags to `Cargo.toml`.
The `cookie` crate is re-exported through axum-extra — you do not add it separately.

```toml
[dependencies]
axum = "0.8"
axum-extra = { version = "0.10", features = ["cookie", "cookie-signed", "cookie-private"] }
tower-cookies = "0.11"       # required by axum-extra cookie features
```

| Feature flag     | Enables                                    |
| ---------------- | ------------------------------------------ |
| `cookie`         | `CookieJar` — unsigned, plain-text cookies |
| `cookie-signed`  | `SignedCookieJar` — HMAC-signed cookies    |
| `cookie-private` | `PrivateCookieJar` — AES-encrypted cookies |

Enable only the features you actually use. Each adds a dependency:
`cookie-signed` pulls in `hmac` + `sha2`; `cookie-private` pulls in `aes-gcm`.

---

## Key Generation

Signed and private cookie jars require a cryptographic key. Generate one at
application startup and never hardcode it.

```rust
use axum_extra::extract::cookie::Key;

// Generate a random 64-byte key (suitable for both signed and private jars).
let key = Key::generate();

// Persist / restore a key from a base64 string (e.g. loaded from env/config).
let key = Key::from(
    base64::Engine::decode(
        &base64::engine::general_purpose::URL_SAFE,
        "your_base64_encoded_64_byte_key_here__________",
    )
    .expect("valid base64"),
);

// Derive the key from application state via FromRef (see Complete Example below).
```

**Why 64 bytes?** The key is split in half: the first 32 bytes feed the HMAC
signing key, the remaining 32 bytes feed the AES-GCM encryption key. This single
key works for both `SignedCookieJar` and `PrivateCookieJar`.

---

## CookieJar (Unsigned)

`CookieJar` reads and writes plain-text cookies with no cryptographic protection.
Use it for non-sensitive preferences like theme selection or language.

**Why unsigned?** Plain cookies are the smallest and fastest to process. They are
appropriate when the client can safely know (and modify) the value.

```rust
use axum::{extract::FromRequestParts, http::request::Request, response::IntoResponseParts};
use axum_extra::extract::CookieJar;

// --- Reading cookies ---
async fn read_theme(jar: CookieJar) -> String {
    // .get() returns Option<Cookie>
    jar.get("theme")
        .map(|c| c.value().to_owned())
        .unwrap_or_else(|| "light".into())
}

// --- Adding / updating cookies ---
async fn set_theme(jar: CookieJar) -> impl IntoResponseParts {
    let cookie = jar
        .add(
            cookie::Cookie::build(("theme", "dark"))
                .path("/")
                .http_only(true)
                .max_age(cookie::Duration::days(365))
                .finish(),
        );
    // IntoResponseParts lets you return cookies alongside any body response.
    (cookie, "Theme set")
}

// --- Removing cookies ---
async fn clear_theme(jar: CookieJar) -> impl IntoResponseParts {
    let cookie = jar.remove(cookie::Cookie::from("theme"));
    (cookie, "Theme cleared")
}
```

### CookieJar as Extractor

`CookieJar` is an extractor — it pulls cookies from the incoming `Cookie` header.
It also implements `IntoResponseParts`, so returning it (or the result of
`jar.add()` / `jar.remove()`) sets the `Set-Cookie` header on the response.

**Why separate extractor and response?** Axum's architecture treats request
extraction and response modification as distinct phases. `CookieJar` participates
in both: extraction reads `Cookie`, `IntoResponseParts` writes `Set-Cookie`.

---

## SignedCookieJar (HMAC-Signed)

Signed cookies are tamper-evident but **not encrypted**. The value is visible to
the client but any modification invalidates the HMAC signature.

**Why sign?** Signing prevents clients from forging cookie values. A session ID
or user role stored in a signed cookie cannot be tampered with.

```rust
use axum_extra::extract::CookieJar;
use axum_extra::extract::cookie::SignedCookieJar;

async fn login(mut jar: SignedCookieJar) -> impl IntoResponseParts {
    // The jar must be mutable to add/remove cookies.
    let jar = jar.add(cookie::Cookie::new("session_id", "abc123"));
    (jar, "Logged in")
}

async fn read_session(jar: SignedCookieJar) -> String {
    // Returns None if the signature is invalid or the cookie is missing.
    jar.get("session_id")
        .map(|c| c.value().to_owned())
        .unwrap_or_else(|| "no session".into())
}

async fn logout(mut jar: SignedCookieJar) -> impl IntoResponseParts {
    let jar = jar.remove(cookie::Cookie::from("session_id"));
    (jar, "Logged out")
}
```

**How it works:** The value is serialized as `<base64_hmac>.<base64_value>`. The
server recomputes the HMAC on each request; mismatched signatures cause the jar
to ignore the cookie silently (returns `None`).

---

## PrivateCookieJar (AES-Encrypted)

Private cookies are both signed and encrypted. The value is unreadable to the
client and tampering is detected.

**Why encrypt?** Use private cookies for sensitive data such as auth tokens,
personal preferences, or anything the client must not see.

```rust
use axum_extra::extract::CookieJar;
use axum_extra::extract::cookie::PrivateCookieJar;

async fn save_token(mut jar: PrivateCookieJar) -> impl IntoResponseParts {
    let jar = jar.add(cookie::Cookie::new("auth_token", "super_secret_jwt"));
    (jar, "Token saved")
}

async fn read_token(jar: PrivateCookieJar) -> String {
    jar.get("auth_token")
        .map(|c| c.value().to_owned())
        .unwrap_or_else(|| "no token".into())
}
```

**How it works:** The value is encrypted with AES-256-GCM, then base64-encoded.
The `Key` provides both the HMAC and AES sub-keys. Decryption fails silently on
tampered cookies, returning `None`.

---

## Cookie Options

Control cookie behavior via `Cookie::build()`:

```rust
use cookie::{Cookie, SameSite, Duration, time::OffsetDateTime};

let cookie = Cookie::build(("user_pref", "en"))
    .path("/")                    // sent on every path (default is current path)
    .domain("example.com")        // restrict to this domain (and subdomains)
    .secure(true)                 // only sent over HTTPS
    .http_only(true)              // invisible to JavaScript
    .max_age(Duration::days(30))  // expires in 30 days
    .same_site(SameSite::Strict)  // CSRF protection
    .expires(                      // absolute expiry (alternative to max_age)
        OffsetDateTime::now_utc() + Duration::days(30),
    )
    .finish();
```

| Option      | Purpose                                                    |
| ----------- | ---------------------------------------------------------- |
| `path`      | URL path prefix the cookie is sent for                     |
| `domain`    | Domain the cookie belongs to (use carefully — avoid `"."`) |
| `secure`    | Cookie is only sent over HTTPS                             |
| `http_only` | Cookie is not accessible via `document.cookie`             |
| `max_age`   | Relative lifetime in seconds                               |
| `expires`   | Absolute expiry `OffsetDateTime`                           |
| `same_site` | `Strict`, `Lax`, or `None` — controls cross-site sending   |

**Why set `secure` and `http_only`?** `secure` prevents cookies from leaking
over plaintext HTTP. `http_only` blocks XSS attacks from stealing cookies.
Always set both for auth-related cookies.

---

## Complete Working Example

This demonstrates all three jar types, `Key` management, and `FromRef`.

```rust
use axum::{
    Router,
    routing::{get, post},
    extract::FromRef,
};
use axum_extra::extract::{CookieJar, cookie::{Key, SignedCookieJar, PrivateCookieJar}};

#[derive(Clone)]
struct AppState {
    key: Key,
}

// Derive Key from AppState so the signed/private jars can extract it.
impl FromRef<AppState> for Key {
    fn from_ref(state: &AppState) -> Self {
        state.key.clone()
    }
}

#[tokio::main]
async fn main() {
    let key = Key::generate();
    let state = AppState { key };

    let app = Router::new()
        .route("/", get(read_all))
        .route("/set", post(set_all))
        .route("/clear", post(clear_all))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn read_all(
    jar: CookieJar,
    signed: SignedCookieJar,
    private: PrivateCookieJar,
) -> String {
    format!(
        "plain theme={:?} | signed session={:?} | private token={:?}",
        jar.get("theme").map(|c| c.value().to_string()),
        signed.get("session_id").map(|c| c.value().to_string()),
        private.get("auth_token").map(|c| c.value().to_string()),
    )
}

async fn set_all(
    jar: CookieJar,
    mut signed: SignedCookieJar,
    mut private: PrivateCookieJar,
) -> impl axum::response::IntoResponse {
    let jar = jar.add(cookie::Cookie::new("theme", "dark"));
    let signed = signed.add(cookie::Cookie::new("session_id", "user_42"));
    let private = private.add(cookie::Cookie::new("auth_token", "jwt_payload"));
    (jar, signed, private, "All cookies set")
}

async fn clear_all(
    jar: CookieJar,
    mut signed: SignedCookieJar,
    mut private: PrivateCookieJar,
) -> impl axum::response::IntoResponse {
    let jar = jar.remove(cookie::Cookie::from("theme"));
    let signed = signed.remove(cookie::Cookie::from("session_id"));
    let private = private.remove(cookie::Cookie::from("auth_token"));
    (jar, signed, private, "All cookies cleared")
}
```

**Why `FromRef`?** `SignedCookieJar` and `PrivateCookieJar` need a `Key` to
verify/decrypt. `FromRef` bridges your application state to the extractor system,
so Axum can automatically pull the key out of your state type.

---

## Cookie Patterns

### Session Management with Signed Cookies

Store a server-issued session ID in a signed cookie. The signature prevents
clients from forging session IDs.

```rust
async fn create_session(mut jar: SignedCookieJar) -> impl IntoResponseParts {
    let session_id = uuid::Uuid::new_v4().to_string();
    let cookie = cookie::Cookie::build(("sid", session_id))
        .http_only(true)
        .secure(true)
        .same_site(cookie::SameSite::Strict)
        .max_age(cookie::Duration::hours(24))
        .finish();
    let jar = jar.add(cookie);
    (jar, "Session created")
}
```

### User Preferences with Plain Cookies

Theme, language, and other non-sensitive choices go in unsigned cookies.

```rust
async fn set_lang(jar: CookieJar) -> impl IntoResponseParts {
    let jar = jar.add(
        cookie::Cookie::build(("lang", "ja"))
            .path("/")
            .max_age(cookie::Duration::days(365))
            .finish(),
    );
    (jar, "Language set")
}
```

### Sensitive Data with Private Cookies

Auth tokens, feature flags tied to paid tiers, or personal data should be
encrypted so the client cannot read or forge them.

```rust
async fn store_api_key(mut jar: PrivateCookieJar) -> impl IntoResponseParts {
    let jar = jar.add(
        cookie::Cookie::build(("api_key", "sk_live_abc123"))
            .http_only(true)
            .secure(true)
            .path("/api")
            .finish(),
    );
    (jar, "API key stored securely")
}
```

### Cookie-Based Authentication

Combine a signed session cookie with middleware or per-handler checks.

```rust
async fn protected_route(signed: SignedCookieJar) -> String {
    match signed.get("user_id") {
        Some(c) => format!("Welcome, {}", c.value()),
        None => "Unauthorized".to_string(),
    }
}
```

### Cookie Consent Handling

Track whether the user has accepted cookies before setting non-essential ones.

```rust
async fn set_analytics(jar: CookieJar) -> impl axum::response::IntoResponse {
    let consented = jar.get("consent")
        .map(|c| c.value() == "yes")
        .unwrap_or(false);

    if consented {
        let jar = jar.add(cookie::Cookie::new("analytics_id", "ga_123"));
        (jar, "Analytics enabled").into_response()
    } else {
        "Consent required".into_response()
    }
}
```

---

## Signed vs Private: When to Use Which

| Concern              | `CookieJar` (unsigned) | `SignedCookieJar`  | `PrivateCookieJar` |
| -------------------- | ---------------------- | ------------------ | ------------------ |
| Client can read      | Yes                    | Yes                | No                 |
| Client can forge     | Yes                    | No                 | No                 |
| Client can modify    | Yes                    | No                 | No                 |
| CPU overhead         | None                   | Low (HMAC)         | Medium (AES-GCM)   |
| Cookie size overhead | None                   | ~40 bytes          | ~50 bytes          |
| Key required         | No                     | Yes                | Yes                |
| Use for              | Preferences, consent   | Session IDs, roles | Tokens, PII        |

**Rule of thumb:** Use `CookieJar` for data the client may freely read and change.
Use `SignedCookieJar` when the client may read but not write the value.
Use `PrivateCookieJar` when the client must not read the value at all.

---

## Common Pitfalls

1. **Forgetting `FromRef`** — `SignedCookieJar` and `PrivateCookieJar` require a
   `Key` from state. If you call `.with_state(state)` but do not implement
   `FromRef<AppState> for Key`, extraction panics at runtime with a clear error.

2. **Key mismatch across restarts** — If you regenerate the key on every process
   restart, all existing signed/private cookies become invalid. Persist the key
   (env var, config file, secret manager) and reuse it across deployments.

3. **Mutability** — `jar.add()` and `jar.remove()` consume the jar and return a
   new one. You must use `mut jar` (or rebind the variable) to chain additions.

4. **Multiple jars in one handler** — You can extract all three jar types in the
   same handler. Axum reads the `Cookie` header once and distributes it to each
   jar. On the response side, each jar's `Set-Cookie` headers are combined.

5. **Cookie size limits** — Browsers enforce a 4 KB per-cookie limit. Encrypted
   cookies are ~33% larger than plain values due to base64 encoding. Do not
   store large payloads in cookies; store an ID and look up server-side data.

6. **`domain` edge case** — Setting `domain("example.com")` makes the cookie
   available to `sub.example.com` too. Omit `domain` to restrict to the exact
   origin. Never set `domain(".example.com")` — the leading dot is deprecated.

7. **`SameSite` defaults** — Modern browsers default to `Lax`. If you need
   cross-site cookies (e.g., OAuth callbacks), you must explicitly set
   `same_site(SameSite::None)` and also set `secure(true)`.
