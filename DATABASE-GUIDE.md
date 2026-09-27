# LANBox — Database Guide

There is no database — by decision (ADR-003), not by omission. This guide
records what stands in for a schema, which guideline sections are N/A and
why, and the path back if persistence is ever needed.

## A. Database selection

None. Rationale: the data is files, the query is `readdir`, there are no
relations, no concurrent writers beyond the transfer semaphore, and one
operator. A schema for four rows of state would be overhead with zero
reads to optimize.

## B. Schema design (state inventory instead of ERD)

| State | Where | Fields |
|---|---|---|
| Served files | Filesystem root (`--dir`) | name, type, size (from OS) |
| Server identity | PID file `~/.local/share/lanbox/lanbox.pid` | pid, address, dir |
| Session | Process memory | per-boot token, PIN flag |
| Shares (V2) | Process memory map | share token, path, expiry, PIN flag |

Root conventions: served names pass through unchanged (no renaming);
anything failing the path gate is rejected, never sanitized into a new
name. PID file is rewritten on boot, removed on clean shutdown, treated
as stale if the PID is dead.

## C. Data types

N/A (no columns). Working equivalents: sizes are `int64` bytes, times are
server-clock Unix seconds (expiry), tokens are 32 random bytes hex-encoded.

## D. Indexing strategy

N/A (no queries to index). Equivalent performance control: directory
listings paginate past 1000 entries (cursor `offset`, see API-GUIDE), and
lookups are single `stat` calls, not scans.

## E. Relationships

N/A (no tables, no joins). Share-to-file is a pointer, not a relation:
deleting the file invalidates the share (404), deleting the share never
touches the file.

## F. Performance

Stream everything (`io.Copy`); never load a whole file. Cap concurrency at
4 (ADR-006). No cache layer — the OS page cache is the cache. Pre-write
disk check returns 507 before a partial lands.

## G. Migration strategy

N/A (no schema to version). If shares ever need to survive restarts: add
SQLite via a new ADR, one table (`shares`), WAL mode, zero-downtime
trivially satisfied (single writer, migration at boot).

## H. Monitoring & maintenance

Watch disk space (the 507 path must trigger before the disk fills, not
after). No VACUUM/ANALYZE (no engine). PID file hygiene: stale detection
on `status`. Logs rotate via the OS; the binary never manages retention.
