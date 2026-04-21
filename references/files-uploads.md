# Axum 0.8 File Handling & Uploads Reference

## Multipart Form Handling

Axum uses the `axum::extract::Multipart` extractor to parse `multipart/form-data` requests. It consumes the request body stream, so the extractor takes ownership — you cannot combine it with another body extractor on the same handler.

### Basic Multipart Extraction

```rust
use axum::{extract::Multipart, response::IntoResponse, http::StatusCode};
use axum_extra::extract::MultipartError;

async fn upload(mut multipart: Multipart) -> Result<impl IntoResponse, (StatusCode, String)> {
    while let Some(field) = multipart.next_field().await
        .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?
    {
        // Each field has these properties:
        let _name: Option<String>         = field.file_name();      // Original filename from Content-Disposition
        let _content_type: Option<String> = field.content_type();   // MIME type from Content-Type header
        let _headers: &axum::http::HeaderMap = field.headers();     // Raw headers on this part

        // Read entire field into memory as raw bytes:
        let data = field.bytes().await
            .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?;

        // Read as UTF-8 text (returns error if bytes are not valid UTF-8):
        let text = field.text().await
            .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?;

        // Stream chunks manually (see Streaming section below):
        // while let Some(chunk) = field.chunk().await.map_err(...)? { ... }
    }
    Ok(StatusCode::OK)
}
```

**Why it works:** The `Multipart` extractor reads from the request body using hyper's streaming API. Each call to `next_field()` parses the next MIME boundary segment. `bytes()` consumes all remaining chunks into a contiguous `Bytes` buffer; `text()` does the same but additionally validates UTF-8. `chunk()` yields one chunk at a time without buffering the entire field, which is critical for large uploads.

## File Upload Patterns

### Single File Upload

```rust
use axum::{extract::Multipart, Json};
use serde_json::{json, Value};

async fn upload_single(mut multipart: Multipart) -> Result<Json<Value>, (StatusCode, String)> {
    let field = multipart
        .next_field().await
        .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?
        .ok_or((StatusCode::BAD_REQUEST, "no field provided".into()))?;

    let filename = field.file_name()
        .ok_or((StatusCode::BAD_REQUEST, "missing filename".into()))?
        .to_string();
    let content_type = field.content_type()
        .ok_or((StatusCode::BAD_REQUEST, "missing content-type".into()))?
        .to_string();
    let data = field.bytes().await
        .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?;

    // Save to disk (see filesystem section below for production patterns)
    tokio::fs::write(format!("/tmp/uploads/{filename}"), &data).await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    Ok(Json(json!({ "filename": filename, "size": data.len(), "content_type": content_type })))
}
```

**Why it works:** `file_name()` reads the `filename` parameter from the `Content-Disposition` header of the multipart field. It returns `None` for non-file fields, so we validate its presence to enforce a file upload.

### Multiple File Upload

```rust
use axum::{extract::Multipart, Json};
use serde_json::{json, Value};

async fn upload_many(mut multipart: Multipart) -> Result<Json<Value>, (StatusCode, String)> {
    let mut saved = Vec::new();
    while let Some(field) = multipart.next_field().await
        .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?
    {
        let Some(filename) = field.file_name() else { continue };
        let filename = filename.to_string();
        let data = field.bytes().await
            .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?;
        tokio::fs::write(format!("/tmp/uploads/{filename}"), &data).await
            .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
        saved.push(json!({ "filename": filename, "size": data.len() }));
    }
    Ok(Json(json!({ "files": saved })))
}
```

### Mixed Form Fields (Text + Files)

```rust
use axum::{extract::Multipart, Json};
use serde_json::{json, Value};

async fn upload_mixed(mut multipart: Multipart) -> Result<Json<Value>, (StatusCode, String)> {
    let mut description = String::new();
    let mut files = Vec::new();

    while let Some(field) = multipart.next_field().await
        .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?
    {
        let name = field.name().unwrap_or_default().to_string();
        if name == "description" {
            description = field.text().await
                .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?;
        } else if let Some(filename) = field.file_name() {
            let data = field.bytes().await
                .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?;
            files.push(json!({ "filename": filename, "size": data.len() }));
        }
    }
    Ok(Json(json!({ "description": description, "files": files })))
}
```

**Why it works:** `field.name()` returns the form field name (the part before `=` in `Content-Disposition: form-data; name="..."`). File fields also have a `filename` attribute; text-only fields do not. Use `file_name()` being `None` vs `Some` to distinguish them.

### Streaming Large File Uploads with `chunk()`

```rust
use axum::{extract::Multipart, http::StatusCode};
use tokio::io::AsyncWriteExt;

async fn upload_streaming(mut multipart: Multipart) -> Result<StatusCode, (StatusCode, String)> {
    while let Some(field) = multipart.next_field().await
        .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?
    {
        let Some(filename) = field.file_name() else { continue };
        let path = format!("/tmp/uploads/{filename}");
        let mut file = tokio::fs::File::create(&path).await
            .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

        // Read one chunk at a time — avoids buffering the entire file in memory
        while let Some(chunk) = field.chunk().await
            .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?
        {
            file.write_all(&chunk).await
                .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
        }
    }
    Ok(StatusCode::OK)
}
```

**Why it works:** `chunk()` returns `Some(Bytes)` for each data segment between MIME boundaries, and `None` when the field is complete. This lets you write each chunk to disk immediately, keeping peak memory usage proportional to chunk size rather than total file size. The default chunk size comes from hyper's framing (typically 8 KiB–64 KiB).

### Saving to Filesystem with `tokio::fs`

```rust
use std::path::Path;
use axum::extract::Multipart;

async fn save_uploads(
    mut multipart: Multipart,
    base_dir: &Path,
) -> Result<(), Box<dyn std::error::Error>> {
    tokio::fs::create_dir_all(base_dir).await?;

    while let Some(field) = multipart.next_field().await? {
        let Some(filename) = field.file_name() else { continue };
        // Sanitize: strip directory components to prevent path traversal
        let safe_name = std::path::Path::new(filename)
            .file_name()
            .ok_or("invalid filename")?;
        let dest = base_dir.join(safe_name);
        let data = field.bytes().await?;
        tokio::fs::write(&dest, &data).await?;
    }
    Ok(())
}
```

**Why `tokio::fs`:** Standard `std::fs` operations block the async executor thread. `tokio::fs` spawns blocking I/O onto Tokio's blocking thread pool, preventing the executor from stalling.

## Upload Size Limits

Axum's default body limit is ~2 MB. Exceeding it returns a `413 Payload Too Large` response before your handler runs.

### Global Limit

```rust
use axum::{routing::post, Router, body::Body, extract::DefaultBodyLimit};

let app = Router::new()
    .route("/upload", post(upload_handler))
    // Raise to 50 MB for the entire application
    .layer(DefaultBodyLimit::max(50 * 1024 * 1024));
```

**Why it works:** `DefaultBodyLimit` is implemented as a tower layer that wraps the body stream with a size-checking wrapper. When the accumulated bytes exceed the limit, it immediately returns a `413` response and drops the connection.

### Per-Route Limit

```rust
use axum::{routing::{get, post}, Router};
use axum::extract::DefaultBodyLimit;

let app = Router::new()
    .route("/upload", post(upload_handler))
    // Override limit for this single route (10 MB)
    .route("/upload", post(upload_handler)
        .layer(DefaultBodyLimit::max(10 * 1024 * 1024)))
    .route("/small-upload", post(small_handler))
    .route("/api/data", get(data_handler));
    // /api/data retains the default ~2 MB limit
```

### Disable Limit (use with caution)

```rust
.layer(DefaultBodyLimit::disable())
// WARNING: an attacker can exhaust server memory by sending an unbounded body
```

## Multipart Error Handling

The `Multipart` extractor uses `axum::extract::rejection::MultipartRejection` as its rejection type. Common inner errors include `MultipartError::InvalidBoundary` (malformed request) and `MultipartError::IncompleteField` (client disconnected mid-upload).

```rust
use axum::{
    extract::Multipart,
    http::StatusCode,
    response::{IntoResponse, Response},
};

enum AppError {
    Multipart(axum::extract::rejection::MultipartRejection),
    Io(std::io::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::Multipart(r) => {
                // MultipartRejection contains the underlying MultipartError
                (StatusCode::BAD_REQUEST, format!("multipart error: {r}"))
            }
            AppError::Io(e) => {
                (StatusCode::INTERNAL_SERVER_ERROR, format!("io error: {e}"))
            }
        };
        (status, message).into_response()
    }
}

// Return Result<_, AppError> to get automatic rejection conversion
async fn safe_upload(multipart: Result<Multipart, AppError>) -> Response {
    match multipart {
        Ok(mut mp) => {
            while let Ok(Some(field)) = mp.next_field().await {
                // process field...
            }
            StatusCode::OK.into_response()
        }
        Err(e) => e.into_response(),
    }
}
```

**Why this pattern:** By accepting `Result<Multipart, Rejection>` in the handler signature, the extractor never rejects early — your handler always runs and can return a structured error response instead of relying on Axum's default plain-text rejection.

## Static File Serving

### ServeDir for Directories

```rust
use axum::{routing::get_service, Router};
use tower_http::services::{ServeDir, ServeFile};

let app = Router::new()
    .nest_service("/static", ServeDir::new("assets"));
// GET /static/style.css → serves ./assets/style.css
// GET /static/js/app.js  → serves ./assets/js/app.js
```

**Why `nest_service` vs `route_service`:** `nest_service` strips the matched prefix (`/static`) before passing the remaining path to the service. `route_service` does not strip, so `ServeDir` would need its base path set to include the prefix. For directory serving, always use `nest_service`.

### ServeFile for Individual Files

```rust
use axum::routing::get_service;
use tower_http::services::ServeFile;

let app = Router::new()
    .route_service("/favicon.ico", ServeFile::new("assets/favicon.ico"));
```

### SPA Fallback Pattern

Serve static assets from a directory, but fall back to `index.html` for unknown paths (required by client-side routers like React Router or Vue Router):

```rust
use tower_http::services::{ServeDir, ServeFile};

let app = Router::new().nest_service(
    "/",
    ServeDir::new("dist")
        .fallback(ServeFile::new("dist/index.html")),
);
// GET /about     → dist/about (if file exists), else dist/index.html
// GET /static.js → dist/static.js
```

**Why it works:** `ServeDir` tries to find the file in `dist/`. If the file does not exist, it delegates to the fallback `ServeFile`, which returns `dist/index.html` with a `200` status. This lets the client-side router handle URL paths.

### Pre-compressed Files

If you have pre-compressed `.gz` files alongside your assets, enable automatic negotiation:

```rust
ServeDir::new("assets").precompressed_gzip();
// When a client sends Accept-Encoding: gzip and assets/style.css.gz exists,
// ServeDir serves the .gz file with Content-Encoding: gzip automatically.
```

## File Download

### Setting Content-Disposition Headers

```rust
use axum::{
    http::{header, StatusCode},
    response::{IntoResponse, Response},
};

async fn download_file() -> impl IntoResponse {
    let data = tokio::fs::read("files/report.pdf").await.unwrap();
    let headers = [
        (header::CONTENT_TYPE, "application/pdf".to_string()),
        // "attachment" prompts a download; "inline" previews in-browser
        (
            header::CONTENT_DISPOSITION,
            r#"attachment; filename="report.pdf""#.to_string(),
        ),
    ];
    (StatusCode::OK, headers, data)
}
```

### Streaming File Responses with `AsyncReadBody` (axum-extra)

```rust
use axum::response::IntoResponse;
use axum_extra::body::AsyncReadBody;
use tokio::fs::File;

async fn stream_download() -> impl IntoResponse {
    let file = File::open("large-video.mp4").await.unwrap();
    let body = AsyncReadBody::new(file);
    // AsyncReadBody streams from the file without loading it entirely into memory
    ([(axum::http::header::CONTENT_TYPE, "video/mp4")], body)
}
```

**Why `AsyncReadBody`:** It wraps any `tokio::io::AsyncRead` into an axum-compatible body. The file is read on-demand as chunks are sent to the client, so memory usage stays constant regardless of file size. This is essential for serving large media files.

### FileStream from axum-extra

```rust
use axum_extra::response::FileStream;

async fn file_stream_download() -> impl IntoResponse {
    FileStream::new(tokio::fs::File::open("data.csv").await.unwrap())
}
```

**Why FileStream over raw AsyncReadBody:** `FileStream` automatically sets `Content-Type` by inspecting the file extension via the `mime_guess` crate, and sets `Content-Length` from file metadata. Use it when you want sensible defaults without manual header configuration.

### Attachment Header Helper (axum-extra)

```rust
use axum_extra::response::Attachment;

async fn download_with_attachment() -> impl IntoResponse {
    let file = tokio::fs::File::open("report.pdf").await.unwrap();
    // Sets Content-Disposition: attachment; filename="report.pdf"
    // and auto-detects Content-Type
    Attachment::new(file).filename("report.pdf")
}
```

## Content-Type Handling

`tower_http::services::ServeDir` and `ServeFile` automatically detect MIME types from file extensions using the `mime_guess` crate. This lookup happens at serve time and requires no configuration.

To override the detected content type:

```rust
use axum::routing::get_service;
use tower_http::services::ServeFile;
use http::header::CONTENT_TYPE;

// Override: serve a .bin file as application/octet-stream
let service = ServeFile::new("data/export.bin")
    .with_headers([
        (CONTENT_TYPE, "application/octet-stream".parse().unwrap()),
    ]);
```

**Why you might override:** Some CDNs or proxies incorrectly sniff content types. Explicitly setting `Content-Type` prevents them from misinterpreting binary data as text or HTML (which can cause XSS in reflected-file attacks).

## Security Considerations

### Path Traversal Prevention

Never use user-supplied filenames directly in filesystem paths:

```rust
use std::path::Path;

fn safe_filename(raw: &str) -> Option<&str> {
    // Extract only the final component — strips "../" and "/etc/" etc.
    Path::new(raw).file_name()?.to_str()
}

// DANGEROUS:
//   let path = format!("/uploads/{}", user_filename);  // "../secrets" escapes!

// SAFE:
//   let safe = safe_filename(user_filename).ok_or("bad name")?;
//   let path = base_dir.join(safe);
```

**Why `file_name()` works:** It returns only the final path component. `Path::new("../../etc/passwd").file_name()` yields `Some("passwd")`, and joining that onto your base directory produces `/uploads/passwd` — safely contained.

### File Size Limits

Always set `DefaultBodyLimit::max()` appropriate to your use case. Unbounded uploads let attackers exhaust server memory:

```rust
// Images: 10 MB
// Documents: 50 MB
// Videos: 500 MB (and use streaming!)
DefaultBodyLimit::max(10 * 1024 * 1024)
```

### Content-Type Validation

Validate that uploaded files match their declared content type:

```rust
fn is_allowed_content_type(ct: Option<&str>) -> bool {
    ct.is_some_and(|t| {
        matches!(t,
            "image/png" | "image/jpeg" | "image/gif" | "image/webp"
            | "application/pdf"
        )
    })
}

// In your handler:
let ct = field.content_type().unwrap_or_default();
if !is_allowed_content_type(Some(ct)) {
    return Err((StatusCode::UNSUPPORTED_MEDIA_TYPE, "file type not allowed".into()));
}
```

**Why it matters:** Browsers determine file behavior by Content-Type. An uploaded SVG file served as `text/html` enables stored XSS. An uploaded HTML file served inline executes scripts. Always validate server-side — never trust client-supplied content-type headers alone.

## Common Pitfalls

1. **Forgetting `DefaultBodyLimit` —** uploads silently fail with 413. Always configure an explicit limit and surface the error in tests.

2. **Using `std::fs` in async handlers —** blocks the executor thread. Always use `tokio::fs` or `tokio::task::spawn_blocking`.

3. **Calling `bytes()` after `chunk()` (or vice versa) —** these methods consume the field's internal stream. Call only one per field.

4. **Double-extracting `Multipart` and `String` body —** the body stream can only be consumed once. Combine all parsing into the multipart handler.

5. **Missing `nest_service` for `ServeDir` —** using `route_service("/static", ServeDir::new("."))` serves files at paths like `/static/path` but ServeDir looks for `./static/path`. Use `nest_service` to strip the prefix.

6. **Not sanitizing filenames —** user-supplied filenames like `../../../etc/passwd` cause path traversal. Always extract `file_name()` and join onto a fixed base directory.

7. **Serving user uploads with `ServeDir` —** if users can upload `.html` or `.svg` files and those are served from the same `ServeDir`, you have a stored XSS vulnerability. Serve user uploads from a separate path or with forced `Content-Disposition: attachment`.

8. **Ignoring `MultipartRejection` —** the default rejection returns a plain 400 with no detail. Accept `Result<Multipart, MultipartRejection>` in your handler to return structured error JSON.
