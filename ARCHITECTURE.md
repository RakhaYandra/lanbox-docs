# LANBox — Architecture

Single Go binary: HTTP server + embedded vanilla web UI. No DB, no framework.

```
                    LAN
                     │
              ┌──────▼──────┐
              │   LANBox    │
              │    Server   │
              └──────┬──────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
     Browser        CLI        Browser
      Phone       Laptop        Tablet
```

## Layers

```
HTTP (net/http stdlib)
 │
 ├── Router (ServeMux)
 │
 ├── Middleware (auth token/PIN, transfer semaphore, logging)
 │
 ├── Handler (info, list, upload, download, shares)
 │
 ├── Transfer Manager (progress, cancel, rate limit, checksum)
 │
 ├── File Service (browser, path validation)
 │
 └── Filesystem (root dir)
```

CLI (`cobra`): `serve`, `status`, `send`, `receive`, `stop`, `version`.
Web UI (`web/`, `go:embed`): `index.html`, `app.js`, `style.css` — vanilla only.

## Dependency rule

`server → transfer → filesystem → stdlib`. Inner layers never import
`server` or `cobra`. `filesystem` and `auth` are stdlib-only (grep-verified).

## Flows

Upload: `POST multipart → auth → semaphore acquire → validate path →
stream io.Copy to disk → progress → checksum (V2) → release`.
Download: `GET → auth → validate path → stream file → support Range (V2)`.
Disconnect: request context cancels the copy; partial upload removed.

## Concurrency

Goroutine per transfer, streaming I/O, context cancellation.
Semaphore max 4; excess queued with 429. No transfer blocks the server.

## Configuration

Precedence: CLI flags → env (`LANBOX_PORT`, `LANBOX_DIR`, `LANBOX_HOST`)
→ config file → defaults (`~/LANBox`, `8080`, `0.0.0.0`).

## Security

Per-boot token + optional PIN in middleware. Filesystem gate:
`Clean → Resolve → Validate inside Root`; symlink escape rejected.
Future: TLS, mDNS discovery.
