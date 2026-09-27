# LANBox — FSD (Functional Specification Document)

Per-feature behavior. References PRD `FR-xx`; verified by future integration
tests in the `lanbox` repo. Conventions: base `/api/v1`, auth
`?token=` or `Authorization: Bearer <token>`, errors `{"error": msg}` with
400/401/404/429/507. UI language: English.

## FS-01 Serve — `lanbox serve` (FR-01)

Overview: starts the server with one command and prints everything a second
device needs. User flow: 1. run `lanbox serve [--dir ~/LANBox --port 8080
--host 0.0.0.0]`; 2. read localhost URL, LAN IP, port, PIN, QR; 3. stop with
Ctrl+C. Detailed spec: defaults dir `~/LANBox`, port `8080`, host
`0.0.0.0`; boot under 1s; graceful shutdown drains in-flight transfers up
to 10s. Business rules: one server per port; second serve on same port
fails with `address already in use — try --port 8081`. Input: flags + env
(`LANBOX_DIR/PORT/HOST`). Output: console block (URLs, PIN, QR). Validations:
dir must exist or be creatable (else `cannot access <dir>`); port 1-65535.
Edge cases: port clash, dir on full disk (warn at boot), double boot.
Dependencies: none (root feature). Performance: boot < 1s. l10n: paths and
numbers locale-independent. Accessibility: console output is plain text,
screen-reader safe. Related: PRD FR-01, ARCHITECTURE §1/§16, ADR-007.

## FS-02 Browse + info — `GET /files`, `GET /info` (FR-02)

Overview: list directory entries and server identity. User flow: 1. open
URL; 2. see entries; 3. tap folder to navigate (breadcrumb tracks path).
Detailed spec: `GET /api/v1/files?path=` returns `{path, entries: [{name,
type, size}]}` (`type`: `file`|`directory`; `size` omitted for dirs);
empty path means root; `GET /api/v1/info` returns `{name, hostname,
address, port}`. Business rules: listing never follows symlinks escaping
root; dotfiles listed (hidden only if OS-hidden flag, future). Input: path
string. Output: JSON above. Validations: path gate (`..`, absolute,
`%2e`, double-encoding, symlink escape → 400 `Invalid path`). Errors: 401
`Unauthorized — wrong token`, 404 `Not found`. Edge cases: 1000+ entries
(paginate, see API-GUIDE), file deleted mid-list, permission-denied entry
(skipped + counted). Dependencies: FR-01 serve, FR-06 token. Performance:
p95 < 100ms localhost. Accessibility: list is semantic HTML, breadcrumb
navigable by keyboard. Related: PRD FR-02, DESIGN-SYSTEM §5/§6, ADR-001.

## FS-03 Upload — `POST /files/upload` (FR-03)

Overview: stream a file to the server with progress. User flow: 1. tap
`[ Upload Files ]`; 2. pick files; 3. watch bar (`9.8 / 12 MB, 82%, 92
MB/s`); 4. done → entry appears. Detailed spec: multipart, field `file`;
server streams via `io.Copy` (never full file in RAM); progress events at
most 5 Hz. Business rules: max 4 concurrent (5th → 429 `Server busy` +
`Retry-After`); same-name upload overwrites (explicit, documented); 0-byte
files accepted. Input: multipart bytes + target path. Output: `{name, size,
checksum?}` (checksum V2). Validations: path gate, size cap (disk-aware),
401 without token. Errors: 400/401/429/507 `Disk full`. Edge cases: client
disconnect (partial removed, WARN log), name collision, special filenames
(escaped on render). Dependencies: FR-01, FR-06. Performance: near LAN
line-rate; memory flat. Accessibility: `role="progressbar"` + text mirror.
Related: PRD FR-03, DESIGN-SYSTEM §5/§6, SECURITY-GUIDE §C.

## FS-04 Download — `GET /files/download` (FR-04)

Overview: stream a file to the client. User flow: 1. tap row (or checkbox
several → sequential downloads in MVP); 2. file saves; 3. V2 shows
`Verified` after checksum. Detailed spec: `GET
/api/v1/files/download?path=` streams bytes with content-length; V2 honors
`Range` for resume. Business rules: multi-select downloads sequentially in
MVP (parallel later); directory download rejected (400, zip is future).
Input: path (+`Range` V2). Output: binary stream. Validations: path gate,
401. Errors: 400/401/404. Edge cases: file changing mid-download (V2
checksum catches), huge file on small-disk client (client-side concern,
documented). Dependencies: FR-01, FR-06, FR-07 (V2 resume). Performance:
line-rate; 8 concurrent clients no crash. Accessibility: download is a
plain link (keyboard + screen-reader native). Related: PRD FR-04,
API-GUIDE §C, ADR-006.

## FS-05 CLI transfer — `send`, `receive`, `status`, `stop`, `version` (FR-05)

Overview: terminal client for the same server. User flow (`send`): 1. run
`lanbox send photo.zip --to 192.168.1.10:8080`; 2. watch `photo.zip`,
bar, `1.4 GB, 18.2s, 78.4 MB/s`; 3. exit 0. `receive` mirrors download.
`status` prints PID, dir, address, active transfers from the PID file.
`stop` kills via PID. `version` prints build. Business rules: `send`
requires reachable server (else `cannot connect — is lanbox serve
running?`); stale PID file reports `not running` and offers cleanup.
Input: file path, `--to` address, token via env `LANBOX_TOKEN` or flag.
Output: progress lines + summary. Validations: local file must exist and be
readable; address parse. Edge cases: server dies mid-send (partial kept
server-side only after V2 resume), stop with active transfers (warns, then
kills). Dependencies: FR-01 serve + PID file. Performance: same as FS-03/04.
Accessibility: plain-text output, no color-only meaning. Related: PRD FR-05,
ARCHITECTURE §6, ADR-004.

## FS-06 Token + PIN — pairing and access control (FR-06)

Overview: gate every request without accounts. User flow: 1. server boots,
generates token, prints PIN; 2. client connects via QR URL (token embedded)
or types PIN; 3. requests carry token; 4. missing/wrong → 401. Detailed
spec: token 32 random bytes (`crypto/rand`, hex); accepted via `?token=` or
`Bearer`; 6-digit PIN required by default (sent as `X-PIN` or `?pin=`,
verified server-side, constant-time compare); bind address config.
Business rules: token rotates every boot (restart = rotation); never log
the token (strip query before logging); PIN printed once. Input: token/PIN.
Output: 200 or 401 `Unauthorized — wrong token`. Validations: constant-time
compare. Edge cases: QR photographed (documented risk, PIN mitigates);
token in shell history (prefer env). Dependencies: FR-01. Performance:
negligible (one compare). Accessibility: PIN is large-print in console.
Related: PRD FR-06, SECURITY-GUIDE §A/§E, ADR-005.

## FS-07 Resume, checksum, shares, limit — V2 (FR-07)

Overview: survive drops, prove integrity, share temporarily, cap speed.
Flows: drop at 820 MB of 1.4 GB → reconnect sends `Range: bytes=820M-` →
`Resuming from 820 MB...`; after transfer both sides print SHA-256 and
`Verified`; `POST /shares {path, expires: 30m}` returns share URL;
`--limit 20MB/s` throttles via token bucket. Business rules: resume only
when sizes match prefix (else restart); checksum mismatch → error + keep
both copies; expired share → 404; limit applies per transfer. Input/output:
`Range` headers, `{source_hash, dest_hash}`, share JSON. Validations: range
within bounds (416 otherwise), expiry 1m-24h. Edge cases: file modified
between resume attempts, clock skew on expiry (server clock wins).
Dependencies: FS-03/FS-04. Performance: hash streams (no double RAM);
throttle accuracy ±10%. Accessibility: resume/verified announced as text.
Related: PRD FR-07, ARCHITECTURE §7, API-GUIDE §C.

## FS-08 QR pairing (FR-08)

Overview: zero-typing connect. User flow: 1. server prints QR for
`http://<lan-ip>:<port>/?token=...`; 2. phone camera scans; 3. browser opens
authenticated. Detailed spec: QR encodes full URL with token; terminal
fallback prints the URL as text. Business rules: QR regenerated per boot
(token rotation); never include PIN in QR (typed separately). Input: none.
Output: QR block + text URL. Validations: URL must use LAN IP (not
localhost) so other devices routable. Edge cases: font-less terminal
(text fallback), tiny screens (URL wraps). Dependencies: FR-01, FR-06.
Performance: instant. Accessibility: text URL is the accessible
alternative. Related: PRD FR-08, DESIGN-SYSTEM §5.
