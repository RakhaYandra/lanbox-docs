# LANBox — ADR (Architecture Decision Records)

Single file, one record per decision. Status values: Accepted, Proposed,
Deprecated, Superseded.

## ADR-001: stdlib net/http instead of Gin

Status: Accepted.

Context: need an HTTP server for 5 file endpoints (7 with V2 shares). Alternatives: Gin (team knows
it from Pulse), chi, stdlib ServeMux. Factors: binary size, dependency
count, team familiarity, routing needs (flat paths only).

Decision: `net/http` + ServeMux, no framework.

Consequences: positive — zero web deps, small binary, no framework churn;
negative — hand-written middleware, no built-in validation; cost — trivial;
long-term — upgrade-proof while routes stay flat.

Alternatives considered: Gin (why not: overkill for 5 routes, doubles
binary); chi (why not: nicer router but still a dep for no real gain).

Related ADRs: ADR-002 (UI), ADR-004 (CLI).

## ADR-002: vanilla HTML/CSS/JS instead of React

Status: Superseded by ADR-008.

Context: need a phone-first file browser. Alternatives: React+Vite (Pulse
pattern), HTMX, vanilla. Factors: build step, binary embedding, UI
complexity (list + progress only).

Decision: vanilla, embedded via `go:embed`.

Consequences: positive — no build step, single binary serves all;
negative — hand-managed DOM, no component reuse; cost — low at this UI
size; long-term — revisit if UI outgrows list/progress (record new ADR).

Alternatives considered: React (why not: build pipeline for a 2-screen
UI); HTMX (why not: extra dep, marginal gain over fetch).

Related ADRs: ADR-001, DESIGN-SYSTEM.

## ADR-003: no database

Status: Accepted.

Context: need to track served files and server identity. Alternatives:
PostgreSQL (Pulse), SQLite, no DB. Factors: query needs (none — data is
files), ops burden, single-user scope.

Decision: no database. State is the filesystem + PID file + in-memory token.

Consequences: positive — zero ops, nothing to back up beyond files;
negative — no queries, shares cannot persist restarts (V2 in-memory);
cost — none; long-term — if persistent shares needed, SQLite via new ADR.

Alternatives considered: SQLite (why not: a schema for 4 rows of state);
PostgreSQL (why not: server process for a local tool).

Related ADRs: ADR-007, DATABASE-GUIDE.

## ADR-004: cobra for CLI

Status: Accepted.

Context: 6 commands (`serve`, `status`, `send`, `receive`, `stop`,
`version`) with flags. Alternatives: stdlib `flag` manual dispatch, cobra,
urfave/cli. Factors: subcommand ergonomics, help generation, one extra dep.

Decision: cobra.

Consequences: positive — help/usage free, tested dispatch; negative — one
dependency; cost — low; long-term — stable API, negligible churn risk.

Alternatives considered: manual `flag` (why not: 6-way dispatch +
per-command help is hand-rolled bugs); urfave/cli (why not: equal merit,
cobra more familiar).

Related ADRs: ADR-001, FS-05.

## ADR-005: per-boot token + optional PIN, no accounts

Status: Accepted.

Context: need access control without user management. Alternatives:
permanent password, per-boot token + PIN, no auth (trusted LAN), full
accounts. Factors: LAN is semi-trusted, zero-setup goal, no user store.

Decision: 32B random token per boot (query + Bearer) with optional 6-digit
PIN; restart rotates the token.

Consequences: positive — no passwords to manage, no user table, rotation
by restart; negative — token in URL can leak to logs (mitigated: never log
query); QR photo risk (mitigated: PIN); cost — one middleware; long-term —
TLS later via new ADR if needed.

Alternatives considered: accounts/JWT (why not: user store for a
single-user tool); no auth (why not: LAN is not trusted); permanent
password (why not: rotation story worse).

Related ADRs: ADR-003, SECURITY-GUIDE.

## ADR-006: max 4 concurrent transfers, rest queued

Status: Accepted.

Context: concurrent uploads contend for disk and bandwidth. Alternatives:
unlimited, 1-at-a-time, N=4 with 429 queue. Factors: typical home NAS
behavior, predictable memory, simple client retry.

Decision: semaphore 4; excess gets 429 + `Retry-After`; UI shows `Queued`.

Consequences: positive — bounded memory, fair progress; negative — 5th
client waits (documented, retried); cost — one semaphore; long-term —
limit tunable via flag if hardware differs.

Alternatives considered: unlimited (why not: one client can starve disk);
serial (why not: wastes LAN bandwidth).

Related ADRs: FS-03, FS-04.

## ADR-007: single binary via make build, no containers

Status: Accepted.

Context: need to ship to linux/darwin/windows personal machines.
Alternatives: `make build` per-OS binaries, GoReleaser pipeline, Docker
image. Factors: personal use, no registry, no orchestrator on phones.

Decision: `make build` → copy binary → `lanbox serve`. CI builds docs PDFs
only.

Consequences: positive — simplest possible install; negative — manual copy
per machine, no auto-update; cost — a 10-line Makefile; long-term —
GoReleaser when public releases begin (record new ADR).

Alternatives considered: GoReleaser (why not: release infra before first
user); Docker (why not: no daemon on the receiving phone).

Related ADRs: ADR-001, ADR-003.

## ADR-008: web in separate lanbox-web repo with React + Vite

Status: Accepted. Supersedes ADR-002 (embedding part only; vanilla-vs-React
revisited below — ADR-002 stands as history).

Context: UI needs multi-transfer state, progress bars, and error states.
Alternatives: vanilla embedded (ADR-002), React+Vite separate repo (Pulse
pattern), Svelte, HTMX. Factors: team knows React+Vite from pulse-web,
update UI without rebuilding Go, multi-transfer state management.

Decision: separate private repo `lanbox-web` (React + Vite); Go serves the
built static files via `--web-dir`; dev runs Vite `:5173` proxied to API
`:8080`.

Consequences: positive — components (`FileRow`, `ProgressBar`), error
boundaries, `npm run build` testable; UI ships without Go rebuild;
negative — build step + bundle (~100 KB), two repos to clone; cost — Vite
scaffold + proxy config; long-term — bundle stays small (4 endpoints, no
router lib needed).

Alternatives considered: vanilla embedded (why not: hand-managed DOM for 4
concurrent progress states is error-prone; team already pays React cost
on pulse-web); Svelte (why not: no team experience); HTMX (why not:
extra dep, marginal gain over fetch + React state).

Related ADRs: ADR-002, ADR-007, DESIGN-SYSTEM.

## ADR-009: PIN required by default

Status: Accepted. Supersedes ADR-005 (PIN part only — token design stands).

Context: ADR-005 shipped PIN as printed-but-unverified decoration. A secret
the server never checks is worse than none (false confidence). Alternatives:
keep optional-unverified, require by default with `--pin off` escape, drop
PIN entirely.

Decision: PIN verified server-side on every `/api/*` request (`X-PIN` or
`?pin=`, constant-time compare), required by default; `serve --pin off`
disables; `--pin <6-digit>` sets a fixed one. CLI (`--pin`/`LANBOX_PIN`)
and web login screen send it.

Consequences: positive — second factor real; negative — one more value to
pass around (QR URL carries token, PIN typed separately by design);
cost — one middleware branch; long-term — unchanged.

Alternatives considered: unverified PIN (why not: theater); PIN inside QR
URL (why not: photographed-QR risk); drop PIN (why not: token-only auth
on hostile LANs is thin).

Related ADRs: ADR-005, SECURITY-GUIDE.
