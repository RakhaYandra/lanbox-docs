# LANBox — SRS (Software Requirements Specification)

Abridged IEEE 830. Functional IDs (`FR-xx`) defined in PRD; each maps to
FSD functions (`FS-xx`) and a verification method (§7 table).

## 1. Introduction

Scope: LANBox v0.1-v0.3 — serve, browse, upload, download, CLI transfer,
token/PIN, resume/checksum/shares/limit. Out of scope: accounts, database
server, cloud, sync, mDNS, TLS (see PRD §12). Audience: developers and QA
implementing the Go binary. References: PRD, ARCHITECTURE, DESIGN-SYSTEM,
FSD, ADR, API-GUIDE, SECURITY-GUIDE.

## 2. Overall description

Product perspective: single Go process on the operator's device; browser
and CLI clients over plain HTTP on the LAN. No system interfaces beyond
the filesystem (serve dir, PID file) and the LAN socket. Product features
summary: one-command serve, file listing, streaming upload/download with
progress, CLI send/receive, token/PIN gate, QR pairing, V2 resume +
checksum + expiring shares + bandwidth limit. User classes: operator
(runs serve, reads console), browser client (phone-first), CLI client
(technical). Operating environment: linux/amd64, darwin/arm64,
windows/amd64; one shared LAN; no internet.

## 3. Specific requirements

Functional (per PRD FR-01..FR-08, behavior in FSD FS-01..FS-08 — normative
by reference, not repeated here). External interfaces: UI — vanilla
HTML/CSS/JS per DESIGN-SYSTEM (4 endpoints consumed); API — JSON over HTTP
per API-GUIDE (5 file endpoints + 2 share endpoints V2, `{"error"}` convention); hardware — none beyond
a network interface and disk. Performance: list/info p95 < 100ms
localhost; transfers near LAN line-rate; memory flat to 5 GB; 2/4/8
concurrent clients without crash. Design constraints: stdlib-first Go,
no framework, ASCII-only docs, single binary. Quality attributes:
reliability (disconnect-safe, 507 before corruption), availability (serve
until Ctrl+C), security (token/PIN + path gate per SECURITY-GUIDE).

## 4. System features

- SF-01 Serve (FR-01, high): stimulus `lanbox serve` → response console
  block + listening socket; priority: highest (all else depends on it).
- SF-02 Browse (FR-02, high): stimulus `GET /files` → response entry JSON.
- SF-03 Upload (FR-03, high): stimulus multipart POST → response file
  record + progress events.
- SF-04 Download (FR-04, high): stimulus GET → response byte stream.
- SF-05 CLI transfer (FR-05, medium): stimulus `send/receive` → response
  progress + summary; `status/stop/version` as specified.
- SF-06 Access control (FR-06, high): stimulus request without token →
  response 401; traversal → 400.
- SF-07 V2 transfer (FR-07, medium): stimulus drop → resume; completion →
  verified checksums; share CRUD; throttled bytes.
- SF-08 QR (FR-08, medium): stimulus boot → scannable authenticated URL.

## 5. Other nonfunctional requirements

Security: per SECURITY-GUIDE (token/PIN, 5-pattern traversal gate, upload
validation). Performance: per §3. Reliability: per-job isolation via
goroutine + context; partial uploads removed. Maintainability: Go module
with `gofmt`/`go vet` clean, godoc on exports, table-driven tests for
`filesystem` and `auth` (see CODE-STYLE-GUIDE).

## 6. Appendices

Glossary: root (served dir), token (per-boot secret), semaphore (4-slot
transfer gate), share (expiring V2 link), PID file (server identity).
Assumptions: operator's LAN is semi-trusted; clients run modern browsers;
disk space checked before writes. Constraints: no internet at runtime, no
persistent server-side state beyond the filesystem.

## 7. Acceptance criteria

SRS is met when PRD §11 Success Metrics all hold (cross-device transfer,
> 1 GB stable, 4 concurrent, traversal rejected, graceful shutdown, help,
tests, README, single binary).

## 8. Traceability matrix

| FR | FS | Verified by |
|---|---|---|
| FR-01 | FS-01 | Boot test + `--help` |
| FR-02 | FS-02 | curl list + browser nav |
| FR-03 | FS-03 | 1 GB upload, memory flat |
| FR-04 | FS-04 | Download SHA-256 match |
| FR-05 | FS-05 | CLI send/receive roundtrip |
| FR-06 | FS-06 | 401 no-token + 400 traversal matrix |
| FR-07 | FS-07 | Drop/resume + checksum + share expiry + limit |
| FR-08 | FS-08 | QR decode equals token URL |
