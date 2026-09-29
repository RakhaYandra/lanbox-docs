# LANBox — API Guide

## A. Overview

Base URL `http://<lan-ip>:8080/api/v1`. Format: JSON in, JSON out (except
download streams bytes). Auth: `?token=<hex>` or `Authorization: Bearer
<hex>` on every endpoint. Rate limit: 4 concurrent transfers; excess gets
429 + `Retry-After`.

## B. Design principles

REST, plural resources. GET reads, POST uploads/creates, DELETE removes.
Status codes: 200 ok, 201 created, 400 invalid path, 401 unauthorized, 404
not found, 416 bad range, 429 busy, 507 disk full. Errors always
`{"error": "<human words>"}` — plain language, never codespeak.

## C. Endpoints

### GET /api/v1/info

Description: server identity. Auth: token required.

Response (200):
```json
{"name": "LANBox", "hostname": "rakha-legion", "address": "192.168.1.10", "port": 8080}
```

Error (401): `{"error": "Unauthorized — wrong token"}`.

### GET /api/v1/files?path=/Documents

Description: list entries. Auth: token required. Query: `path` (default
`/`, root), `offset` (default 0), `limit` (default 100, max 1000).

Response (200):
```json
{"path": "/Documents", "entries": [
  {"name": "report.pdf", "type": "file", "size": 12485760},
  {"name": "photos", "type": "directory"}
]}
```

Errors: 400 `Invalid path`, 401, 404 `Not found`.

### GET /api/v1/files/download?path=/report.pdf

Description: stream file bytes. Auth: token required. Headers:
`Range: bytes=858993459-` resumes. Response (200): binary with
`Content-Length`; (206) partial for Range. Query `preview=1`: inline
display (`Content-Disposition: inline`, allowlisted types only, 20 MB cap
else 413).

Errors: 400, 401, 404, 413 `Too large to preview`, 416 `Range not satisfiable`.

### POST /api/v1/files/upload

Description: upload one file (multipart field `file`, query `path` =
target dir). Auth: token required.

Response (201): `{"name": "photo.zip", "size": 1476395008}` (V2 adds
`"checksum": "sha256:..."`).

Errors: 400, 401, 429 `Server busy — try again` (+`Retry-After`), 507
`Disk full`.

Rate limit: counts toward the 4 transfer slots.

### DELETE /api/v1/files?path=/report.pdf

Description: delete a file (directories rejected). Auth: token
required. Response (200): `{"deleted": "/report.pdf"}`. Errors: 400, 401,
404.

### POST /api/v1/shares (V2)

Description: create an expiring share link for one file. Auth: server token
required.

Request:
```json
{"path": "/report.pdf", "expires_minutes": 30, "pin_required": false, "allow_upload": false}
```
(`expires_minutes` 1–1440, default 30. `allow_upload` needs a directory
path and turns the share into an expiring inbox.)

Response (201 Created):
```json
{"share_token": "9f2c...", "url": "http://192.168.1.10:8080/api/v1/shares/9f2c...", "expires_at": 1758949200, "pin_required": false}
```

Errors: 400 `Invalid path` / `Invalid expiry`, 401, 404 `Not found`.

### GET /api/v1/shares/:token (V2)

Description: download via share link. Auth: the share token IS the auth
(no server token needed); if `pin_required`, send header `X-Share-PIN`.

Response (200): binary stream (same bytes as the file). Errors: 401
`Wrong share PIN`, 404 `Share expired or not found` (expired and missing
share the same 404 — no expiry oracle).

### POST /api/v1/shares/:token/files

Description: guest upload into an upload share (no server token needed;
share token is in the path). Auth: `X-Share-PIN` when set. Request:
multipart field `file`. Response (201): `{name, checksum}`.

Errors: 400, 401 `Wrong share PIN`, 404, 507.

### GET /api/v1/transfers

Description: list in-flight transfers (registry, not the 4-slot queue —
excess still 429s). Auth: token required. Response (200):
`{"transfers": [{id, name, bytes, total, started_at}]}`.

### DELETE /api/v1/transfers/:id

Description: cancel an in-flight transfer. Auth: token required. Response
(200): `{"cancelled": "<id>"}`. Errors: 401, 404.

### GET /api/v1/history?limit=20&offset=0

Description: persistent transfer log (SQLite, 90-day retention). Auth:
token required. Response (200):
`{"entries": [{id, name, kind, size, duration_ms, sha256, status, finished_at}]}`.

## D. Common patterns

Pagination: `offset`/`limit` on list (cursor = offset). Sorting: name
ascending, directories first. No `include`/`fields` params (no relations,
no partial resources to expand).

## E. Versioning

URL versioning (`/v1/`). Breaking changes ship as `/v2/` with the old
version kept one minor release; deprecations announced in release notes.

## F. Authentication

Token: 32 random bytes, hex, generated per boot, rotated on restart.
Send as `?token=` (QR flow) or `Bearer` header (CLI). The 5 file endpoints
plus `POST /shares` require it. `GET /shares/:token` uses the share token
instead (+ `X-Share-PIN` when set). PIN: 6 digits, required by default,
sent as `X-PIN` header or `?pin=` (`serve --pin off` disables; CLI uses
`--pin` / `LANBOX_PIN`; web login screen stores it in sessionStorage).
No scopes — single privilege level.

## G. Error handling

```json
{"error": "Disk full"}
```

Full code table: 400 invalid path/range/expiry, 401 wrong/missing token
(or wrong share PIN), 404 missing file/share (expired shares also 404),
416 unsatisfiable range, 429 busy (+`Retry-After` header), 507 disk full.
Messages are stable English strings clients may match. Extra example:
`{"error": "Share expired or not found"}`.

## H. Rate limiting & quotas

4 concurrent transfers per server (see ADR-006). Per-transfer `--limit`
(V2) throttles bytes. Response headers on 429: `Retry-After: <seconds>`.
No per-minute quotas — the semaphore is the quota.

## I. Webhooks

None. Clients poll `status` or watch progress inline.

## J. SDK / client libraries

None. `curl` plus `lanbox send`/`receive` are the clients:

```sh
curl -H "Authorization: Bearer $LANBOX_TOKEN" \
  "http://192.168.1.10:8080/api/v1/files?path=/" | jq .
curl -H "Authorization: Bearer $LANBOX_TOKEN" \
  "http://192.168.1.10:8080/api/v1/files/download?path=/report.pdf" -o report.pdf
```

## K. Changelog

| API | Change |
|---|---|
| v1 (initial) | info, list, download, upload, delete |
| v1 + resume (V2) | `Range` on download, `checksum` on upload |
| v1 + shares (V2) | `POST /shares` (expiry 1m–24h, optional PIN), `GET /shares/:token` (share-token auth, expired → 404, server clock wins) |
| v1 + extras | `?preview=1` inline (20 MB cap), `POST /shares/:token/files` upload inbox, `GET/DELETE /transfers[/:id]`, `GET /history`, `serve --drop` |
