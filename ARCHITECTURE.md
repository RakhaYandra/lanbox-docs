# LANBox — Architecture

## 1. System Overview

Single Go process, single static binary: HTTP server for the file API.
The web UI lives in a separate repo (`lanbox-web`, React + Vite) and is
served as static files via `--web-dir`. Embedded SQLite for history only;
no database server, no cloud. Three client
types hit the server directly over the LAN.

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

## 2. Tech Stack

| Layer | Choice | Rejected (why) |
|---|---|---|
| Language | Go 1.27 | — |
| HTTP | `net/http` + ServeMux | Gin: binary stays small, stdlib is enough |
| CLI | cobra | Hand-rolled parsing: error-prone for 6 commands |
| Web UI | React + Vite in `lanbox-web` repo, static build served via `--web-dir` | Vanilla embed: multi-transfer state is hand-managed; team knows React |
| Config | Manual env parsing | Viper: one more dep for 3 vars |
| Rate limit (V2) | `golang.org/x/time/rate` | Custom bucket: clock bugs |
| Hash (V2) | `crypto/sha256` stdlib | — |
| QR (V2) | One small QR lib | Hand-rolled QR: out of scope |
| Docs PDF | pandoc/latex (build-time only) | — |

No database server, no Redis — deliberate difference from Pulse.
Transfer history lives in embedded SQLite (ADR-012).

## 3. Infrastructure

No cloud, no containers. Runtime is the user's own device. Artifacts: the
binary, the served directory (`--dir`, default `~/LANBox`), and a PID file
at `~/.local/share/lanbox/lanbox.pid`. Default port 8080, default bind
`0.0.0.0`, optional `127.0.0.1` for loopback-only.

## 4. Database Schema

No database server — explicit. File data stays as files (nothing to query);
transfer history is the one exception, in embedded SQLite (ADR-012).
Remaining state:

| State | Where | Columns |
|---|---|---|
| Served files | Filesystem root | name, type, size (from `readdir`) |
| Server identity | PID file | pid, address, dir |
| Session | In-memory | per-boot token, PIN (required by default) |
| Shares (V2) | In-memory map | share token, path, expiry, PIN flag |

## 5. API Design

Base `/api/v1`. Auth: `?token=` or `Authorization: Bearer`. Errors always
`{"error": "<human words>"}`.

| Endpoint | Method | Request | Response | Errors |
|---|---|---|---|---|
| `/info` | GET | — | `{name, hostname, address, port}` | 401 |
| `/files?path=` | GET | path | `{path, entries: [{name, type, size}]}` | 400, 401, 404 |
| `/files/download?path=` | GET | path (+`Range` V2) | binary stream | 400, 401, 404 |
| `/files/upload` | POST | multipart | `{name, size, checksum?}` | 400, 401, 429, 507 |
| `/files?path=` | DELETE | path (optional MVP) | `{deleted: path}` | 400, 401, 404 |

## 6. System Components

- Server: router (ServeMux), middleware (auth, semaphore, logging),
  handlers (info, list, upload, download, shares).
- Transfer manager: progress, cancellation, rate limit, checksum.
- File service: browser plus path gate.
- Discovery: LAN IP print, QR, plus mDNS `_lanbox._tcp` advertise
  (`LANBox-<host>-<port>`, TXT presence-only) and `lanbox discover` browse.
- CLI: `serve`, `status`, `send`, `receive`, `stop`, `version`.
- Web UI (separate `lanbox-web` repo, React): consumes the 4 endpoints via
  `fetch` against a configurable base URL; dev runs Vite `:5173` with
  `/api` proxied to `:8080`; prod serves the built files via `--web-dir`;
  CORS from `--web-origin` (default `*` in M1, allowlist in M2 with auth).

## 7. Data Flow Diagram

List: `UI -> GET /files -> auth -> validate path -> readdir -> JSON`.
Upload: `POST multipart -> auth -> semaphore acquire -> validate path ->
io.Copy to disk -> progress -> checksum (V2) -> release`.
Disconnect: `request context cancels -> partial upload removed -> WARN log`.

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

Dependency rule: `server -> transfer -> filesystem -> stdlib`. Inner layers
never import `server` or `cobra`. `filesystem` and `auth` stay stdlib-only
(grep-verified).

## 8. Security Considerations

Threat model: the LAN is not trusted. Layers: per-boot random token,
6-digit PIN required by default, bind-address config. Filesystem gate:
`Clean -> Resolve -> Validate inside Root`, rejecting `..`, absolute
paths, encoded `%2e`, double-encoding, and symlink escape. Uploads: size
cap plus optional dotfile rejection. TLS always on (self-signed, ADR-010).
mDNS live (ADR-011); single-subnet only. Test matrix: each
of the 5 traversal patterns must return 400.

## 9. Scalability Plan

Local-vertical only: concurrent transfers and file sizes, not user count.
No clustering, no horizontal scale (single-user tool by design). Known
ceilings: LAN bandwidth, disk IOPS, 4 transfer slots with the rest queued.

## 10. Performance Targets

File list/info p95 under 100ms on localhost. Transfers run near LAN
line-rate (reference: 1.4 GB in ~18s on a fast link). Memory flat versus
file size. 2/4/8 concurrent clients without crash. Fixtures: 100 MB, 1 GB,
5 GB generated locally, never committed.

## 11. Error Handling & Logging

Codes: 400 invalid path, 401 unauthorized, 404 not found, 429 server busy
(with `Retry-After`), 507 insufficient storage — all with human messages.
Logging via `slog`: `INFO` serve start, upload start/complete; `WARN`
disconnect, queue-full; `ERROR` disk write failure. Stats line: active and
total transferred. No Prometheus for MVP (deliberate, unlike Pulse).

## 12. Third-party Integrations

Runtime: none. Build-time: pandoc/latex Docker image for docs PDFs. V2
adds exactly two: one small QR library and `golang.org/x/time/rate`.

## 13. Deployment Strategy

`make build` produces one binary per OS (linux, darwin, windows); copy it
to the device and run `lanbox serve`. No containers, no compose. GitHub
Actions builds docs PDFs only. Multi-OS GoReleaser pipeline is future.

## 14. Documentation Plan

This repo is canonical: PRD, ARCHITECTURE, DESIGN-SYSTEM, FSD, SRS, ADRs,
and guides live here. Go code gets godoc on exported symbols.
README covers install, usage, and benchmark. Web-to-API contract is guarded
by tests, not prose.

## 15. Concurrency

Goroutine per transfer, streaming I/O, context cancellation. Semaphore max
4; excess queued with 429. No single transfer blocks the server.

## 16. Configuration

Precedence: CLI flags, then env (`LANBOX_PORT`, `LANBOX_DIR`,
`LANBOX_HOST`), then config file, then defaults (`~/LANBox`, `8080`,
`0.0.0.0`).
