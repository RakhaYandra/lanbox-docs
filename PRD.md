# LANBox — PRD (Product Requirements Document)

## 1. Overview / Executive Summary

LANBox is a local-first file transfer tool built in Go for sharing files
directly across devices on the same network, without cloud storage or
external servers. One device runs `lanbox serve`; the rest connect via
browser or CLI. Single static binary plus a separate React web repo
(`lanbox-web`), no account, no internet required.

Companion to Pulse (API monitoring): Pulse is networked SaaS-style Go with
PostgreSQL/Redis/React; LANBox is offline single-binary Go with stdlib HTTP,
streaming I/O, and filesystem security.

## 2. Problem Statement

Moving files between devices needs cloud storage, USB drives, messaging
apps, temporary upload services, cables, or logins. For large files
(e.g. 1.4 GB) that is slow or impractical. LANBox reduces it to: select
file, run server, scan QR, transfer — all inside the LAN.

## 3. Target Users

Primary: developers and technical users moving files between laptop,
desktop, smartphone, and tablet on one local network. Secondary: general
users needing a fast cloud-free transfer.

## 4. Goals & Objectives

1. Serve with one command. 2. Address discoverable without typing IP
   (LAN IP print + QR). 3. Browser lists, uploads, downloads with progress.
4. CLI sends and receives. 5. Multi-file transfer. 6. Token/PIN-gated
   access. 7. Fully offline-capable.

## 5. Feature Requirements

### FR-01 Serve with one command

`lanbox serve [--dir ~/LANBox --port 8080 --host 0.0.0.0]` starts the
server, prints localhost URL, LAN IP, port, PIN, and QR. `lanbox version`
prints the build. `lanbox status` reads the PID file (running PID, dir,
address, active transfers).

### FR-02 Browse files

`GET /api/v1/files?path=` lists entries (`name`, `type`, `size`). Supports
folder navigation. `GET /api/v1/info` returns name, hostname, address, port.

### FR-03 Upload

`POST /api/v1/files/upload` (multipart, streaming via `io.Copy`, never full
file in RAM). Web UI shows progress (bytes, percent, speed).

### FR-04 Download

`GET /api/v1/files/download?path=` streams the file. Multi-select downloads
N files (sequential in MVP, parallel later).

### FR-05 CLI transfer (v0.2.0)

`lanbox send <file>` uploads to a running server with progress, time, and
speed. `lanbox receive` downloads. `lanbox stop` kills via PID file.

### FR-06 Pairing and access control

Per-boot random token (`crypto/rand` 32B, `?token=` + `Authorization:
Bearer`). Optional 6-digit PIN. Bind-address config. Path traversal
rejected (see NFR-04).

### FR-07 Resume, checksum, temporary shares (V2)

Range-based resume (`Resuming from 820 MB...`), SHA-256 verify after
transfer (source == destination), expiring shares (`Expires: 30 minutes`),
optional share PIN, `--limit 20MB/s` bandwidth cap.

### FR-08 QR pairing

Server renders QR for `http://<lan-ip>:<port>/?token=...`. No manual IP
typing.

### NFR-01 Streaming

Stable memory for files > 1 GB (100 MB / 1 GB / 5 GB fixtures, generated,
never committed).

### NFR-02 Concurrency

4 concurrent transfers max, 5th queued (429 + `Retry-After`).

### NFR-03 Robustness

Client disconnect cancels via context, no crash; disk-full returns 507.

### NFR-04 Security

`Clean -> Resolve -> Validate inside Root`; reject `..`, absolute paths,
encoded `%2e`, symlink escape. Consistent JSON errors
(400/401/404/429/507).

### NFR-05 Operability

Graceful shutdown, `slog` lines only (no Prometheus for MVP), single
static binary, `lanbox --help` complete.

## 6. User Stories

- US-01 Serve: as a user I want one command so my device becomes a server.
- US-02 Browse: as a client I want to list files so I can find what to fetch.
- US-03 Upload: as a client I want progress (bytes, percent, speed) so I
  know a 12 MB upload at 82% is still moving.
- US-04 Download: as a client I want one tap to download a file.
- US-05 Multi-file: as a user I want checkboxes so 4 files (128 MB total)
  transfer in one go.
- US-06 QR pairing: as a phone user I want to scan so I never type an IP.

## 7. Acceptance Criteria

- FR-01: given `lanbox serve`, console shows localhost URL, LAN IP, port,
  PIN, QR; Ctrl+C stops cleanly.
- FR-02: given `?path=/Documents`, entries carry correct name/type/size.
- FR-03: given a 1 GB upload, progress runs 0-100% and memory stays flat.
- FR-04: given a download, bytes equal source (SHA-256 match).
- FR-05: given `lanbox send photo.zip`, output shows percent, time, speed.
- FR-06: given no token, server returns 401; given `../`, returns 400.
- FR-07: given a drop at 820 MB of 1.4 GB, reconnect resumes from 820 MB;
  given completion, source and destination hashes match (`Verified`).
- FR-08: given the QR, decoded URL equals `http://<ip>:<port>/?token=...`.

## 8. Scope & Constraints

v0.1: server, web UI, path security, graceful shutdown. v0.2: CLI transfer,
token/PIN. V2: resume, checksum, expiring shares, bandwidth limit.
Constraints: same LAN, single binary, no internet, no database.

## 9. Timeline / Milestones

- M1 v0.1.0: core server + web UI. Done when browser-to-server
  upload/download works.
- M2 v0.2.0: CLI + security. Done when send/receive works and 401/400
  cases are proven.
- M3 v0.3.0: reliability + V2. Done when 4 concurrent transfers, resume,
  checksum, shares, and limit all work.

## 10. Dependencies

Web UI depends on FR-02 to FR-04 APIs. `send`/`receive` depend on a running
`serve` (FR-01). QR depends on the per-boot token (FR-06). Resume depends
on Range support plus checksum (FR-07). `status`/`stop` depend on the PID
file (FR-01/FR-05).

## 11. Success Metrics

Server runs; reachable from another LAN device; files list, upload, and
download; folders browsable; > 1 GB without memory spike; multiple
transfers run; disconnect handled; traversal rejected; graceful shutdown;
`--help` complete; automated tests green; README covers install + usage;
single binary ships.

## 12. Out of Scope

v0.1 excludes: full CLI client, authentication. V2 excludes: mDNS
auto-discovery, TLS (future, not a blocker). Never: user accounts, database
server, cloud storage, sync, collaboration, public internet sharing, file
editing, device management.
