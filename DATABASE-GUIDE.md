# LANBox — Database Guide

One embedded database for one purpose: transfer history (ADR-012, narrowing
ADR-003 — still no database *server*). Everything else remains files,
PID file, and process memory as before.

## A. Database selection

`modernc.org/sqlite` (pure Go — cgo would break the windows/darwin
cross-compile matrix). One file: `~/.local/share/lanbox/transfers.db`.
WAL mode, single writer (the server process). History is best-effort:
if the DB cannot open, the server logs a warning and runs without it.

## B. Schema design

```sql
transfers(
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT, kind TEXT,          -- kind: upload | download
  size INTEGER, duration_ms INTEGER,
  sha256 TEXT, status TEXT,      -- status: completed | cancelled
  finished_at DATETIME
);
CREATE INDEX idx_transfers_finished ON transfers(finished_at DESC);
```

Naming: table plural snake_case, columns snake_case — matching this guide's
conventions from day one. No relations (one table by design).

Non-DB state (unchanged): served files (filesystem root), server identity
(PID file), session token + PIN (memory), shares (memory map).

## C. Data types

Sizes `INTEGER` bytes, times RFC 3339 text (server clock wins), hashes hex
text, `kind`/`status` constrained by code (not ENUM — SQLite has none;
invalid values rejected in `Record` callers by construction).

## D. Indexing strategy

One index on `finished_at DESC` (the only sort order). Reads paginate
(`LIMIT ?, OFFSET ?`, default 20, max 200). No full scans in normal use.

## E. Relationships

N/A (one table, no joins). History rows never reference live files:
deleting a file leaves its history (an honest record, not an oracle).

## F. Performance

Writes are one INSERT per finished transfer plus a retention DELETE —
negligible next to file I/O. No cache layer, no pooling (single writer).

## G. Migration strategy

Schema v1 inline in `Open` (`CREATE TABLE IF NOT EXISTS`). No migrations
yet; when v2 arrives: version table + sequential upgrades at boot,
zero-downtime trivially satisfied (single writer, migration at boot).

## H. Monitoring & maintenance

Retention: 90 days, pruned on every write. Disk: history negligible vs
media files (the 507 path guards the data dir, not the DB). No VACUUM
schedule (WAL + tiny DB); PID-file hygiene and log rotation unchanged.
