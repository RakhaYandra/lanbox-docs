# LANBox — PRD (Product Requirements Document)

Local-first file sharing over the LAN. No cloud, no account, no internet.
One device serves; others connect via browser or CLI.

## FR-01 Serve with one command

`lanbox serve [--dir ~/LANBox --port 8080 --host 0.0.0.0]` starts the server,
prints localhost URL, LAN IP, port, PIN, and QR. `lanbox version` prints the
build. `lanbox status` reads the PID file (running PID, dir, address,
active transfers).

## FR-02 Browse files

`GET /api/v1/files?path=` lists entries (`name`, `type`, `size`).
Supports folder navigation. `GET /api/v1/info` returns name, hostname,
address, port.

## FR-03 Upload

`POST /api/v1/files/upload` (multipart, streaming via `io.Copy`, never full
file in RAM). Web UI shows progress (bytes, percent, speed).

## FR-04 Download

`GET /api/v1/files/download?path=` streams the file. Multi-select downloads
N files (sequential in MVP, parallel later).

## FR-05 CLI transfer (v0.2.0)

`lanbox send <file>` uploads to a running server with progress, time, and
speed. `lanbox receive` downloads. `lanbox stop` kills via PID file.

## FR-06 Pairing and access control

Per-boot random token (`crypto/rand` 32B, `?token=` + `Authorization: Bearer`).
Optional 6-digit PIN. Bind-address config. Path traversal rejected (see NFR-04).

## FR-07 Resume, checksum, temporary shares (V2)

Range-based resume (`Resuming from 820 MB...`), SHA-256 verify after transfer
(source == destination), expiring shares (`Expires: 30 minutes`), optional
share PIN, `--limit 20MB/s` bandwidth cap.

## FR-08 QR pairing

Server renders QR for `http://<lan-ip>:<port>/?token=...`. No manual IP typing.

## NFR

- NFR-01 Streaming: stable memory for files > 1 GB (100 MB / 1 GB / 5 GB fixtures).
- NFR-02 Concurrency: 4 concurrent transfers max, 5th queued (429 + `Retry-After`).
- NFR-03 Robustness: client disconnect cancels via context, no crash; disk-full → 507.
- NFR-04 Security: `Clean → Resolve → Validate inside Root`; reject `..`, absolute,
  encoded `%2e`, symlink escape. Consistent JSON errors (400/401/404/429/507).
- NFR-05 Operability: graceful shutdown, `slog` lines only (no Prometheus for MVP),
  single static binary, `lanbox --help` complete.

## Non-goals

Cloud storage, accounts, database server, sync, collaboration, public sharing,
mDNS auto-discovery (future), TLS (future, not MVP blocker).
