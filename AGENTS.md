# AGENTS.md — lanbox-docs

Docs-only repo. English is canonical. Source of requirements and design;
Go code lives in the future `lanbox` repo.

## Layout

- `PRD.md` — functional + non-functional requirements (`FR-xx`, `NFR-xx`).
- `ARCHITECTURE.md` — layers, flows, dependency rule, config, security.
- `DESIGN-SYSTEM.md` — vanilla web UI contract (tokens, components, API mapping).
- `README.md` — index + PDF + versioning. Keep links in sync with files.
- `.github/workflows/docs-to-pdf.yml` — tag-triggered PDF release.

## Rules

- One concept per edit; keep FR/FS IDs stable (never renumber, only append).
- ASCII diagrams only (must survive xelatex PDF). No emoji in docs.
- PDFs cover `PRD`, `ARCHITECTURE`, `DESIGN-SYSTEM` only. `README`/`AGENTS`
  stay markdown-only (living docs).
- Validate before push:
  `for f in PRD ARCHITECTURE DESIGN-SYSTEM; do test -s "$f.md" || exit 1; done`
  `grep -q "PRD.md" README.md && grep -q "ARCHITECTURE.md" README.md && grep -q "DESIGN-SYSTEM.md" README.md`
- Release: `git tag vX.Y.Z && git push origin vX.Y.Z` → check Actions green →
  check Release has 3 readable PDFs (`test -s pdf/LANBox-*.pdf`).
- Never add Go code, binaries, or fixtures here. FSD/SRS/ADR go here later
  as separate PRs, then join the PDF list.
