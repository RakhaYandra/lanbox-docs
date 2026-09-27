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

Description: stream file bytes. Auth: token required. Headers (V2):
`Range: bytes=858993459-` resumes. Response (200): binary with
`Content-Length`; (206) partial for Range.

Errors: 400, 401, 404, 416 `Range not satisfiable`.

### POST /api/v1/files/upload

Description: upload one file (multipart field `file`, query `path` =
target dir). Auth: token required.

Response (201): `{"name": "photo.zip", "size": 1476395008}` (V2 adds
`"checksum": "sha256:..."`).

Errors: 400, 401, 429 `Server busy — try again` (+`Retry-After`), 507
`Disk full`.

Rate limit: counts toward the 4 transfer slots.

### DELETE /api/v1/files?path=/report.pdf

Description: delete (optional in MVP, server may disable). Auth: token
required. Response (200): `{"deleted": "/report.pdf"}`. Errors: 400, 401,
404.

## D. Common patterns

Pagination: `offset`/`limit` on list (cursor = offset). Sorting: name
ascending, directories first. No `include`/`fields` params (no relations,
no partial resources to expand).

## E. Versioning

URL versioning (`/v1/`). Breaking changes ship as `/v2/` with the old
version kept one minor release; deprecations announced in release notes.

## F. Authentication

Token: 32 random bytes, hex, generated per boot, rotated on restart.
Send as `?token=` (QR flow) or `Bearer` header (CLI). All 5 endpoints
require it. No scopes — single privilege level.

## G. Error handling

```json
{"error": "Disk full"}
```

Full code table: 400 invalid path/range, 401 wrong/missing token, 404
missing file/share, 416 unsatisfiable range, 429 busy (+`Retry-After`
header), 507 disk full. Messages are stable English strings clients may
match.

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
| v1 + shares (V2) | `POST /shares`, `GET /shares/:token` |
