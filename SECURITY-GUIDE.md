# LANBox — Security Guide

Single-user local tool. Sections marked N/A explain why the usual SaaS
control does not apply — not omitted by accident.

## A. Authentication & Authorization

Method: per-boot 32B random token (`?token=` or `Authorization: Bearer`) +
optional 6-digit PIN + bind-address config. No passwords, no sessions, no
2FA/MFA, no RBAC/permission matrix (one operator, one privilege level —
a matrix for one role is theater). Token compared in constant time and
never logged (query stripped before logging). Restart rotates the token.

## B. Data Security

At rest: files served as-is, no encryption (operator's disk is the trust
boundary; OS-level encryption is the documented answer if needed). In
transit: plain HTTP on the LAN; TLS is future work (see ARCHITECTURE),
not MVP. Classification: served files are user-owned; no service-collected
data. PII/GDPR: no accounts, no telemetry, nothing to retain or delete.
Retention/deletion: N/A beyond the filesystem itself.

## C. API Security

Rate limiting: 4-slot transfer semaphore; excess → 429 + `Retry-After`
(see ADR-006). CORS: same-origin UI served by the binary; no wildcard
policy. Input validation: 5-pattern path gate (`..`, absolute, `%2e`,
double-encoding, symlink escape → 400); upload size checked pre-write.
SQL injection: N/A (no database). XSS: filenames HTML-escaped on render.
CSRF: N/A (no cookies/sessions; token is explicit per request). API keys:
the per-boot token is the key — see §E.

## D. Infrastructure Security

Cloud/VPC/DDoS/firewall-rules-as-code: N/A (no public server; the binary
runs on the user's device). Practical hardening: bind `127.0.0.1` for
loopback-only use; prefer trusted Wi-Fi; OS firewall may restrict port
8080. No certificates to manage until TLS lands.

## E. Secrets Management

Only secret is the per-boot token: printed once, never committed, never
logged. CLI reads it from `LANBOX_TOKEN` env or flag (not shell history
where avoidable). Rotation = restart. No vault/KMS for a single ephemeral
secret.

## F. Incident Response

Report via GitHub issue on the docs/code repo. Escalation: maintainer
(single). Post-incident: markdown postmortem in-repo (Pulse pattern).
No on-call for a personal tool.

## G. Security Testing

Per change touching `server`/`filesystem`/`auth`: run the 5-pattern
traversal matrix (all must 400), malicious upload names (special chars,
fake `Content-Type`), missing-token sweep (all endpoints 401). Code review
required on auth/path diffs. No SAST/DAST tooling at this size; `go vet`
plus the matrix is the gate.

## H. Compliance

SOC2/ISO27001: N/A (personal tool, no customer data). Audit logging: `slog`
lines (serve start, upload start/complete, disconnect, denied access) are
the audit trail. Data privacy: nothing collected, nothing to disclose.
