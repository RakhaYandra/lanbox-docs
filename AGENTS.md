# AGENTS.md — lanbox-docs

Docs-only repo. English is canonical. Source of requirements and design;
Go code lives in the future `lanbox` repo.

## Layout

- `PRD.md` — functional + non-functional requirements (`FR-xx`, `NFR-xx`).
- `ARCHITECTURE.md` — layers, flows, dependency rule, config, security.
- `DESIGN-SYSTEM.md` — vanilla web UI contract (tokens, components, API mapping).
- `FSD.md` — per-feature behavior (`FS-01..08` mapped to FR).
- `SRS.md` — abridged IEEE 830 + traceability matrix.
- `ADR.md` — single file, one record per decision (`ADR-001..`).
- `SECURITY-GUIDE.md`, `CODE-STYLE-GUIDE.md`, `DATABASE-GUIDE.md`
  (no-DB rationale), `API-GUIDE.md` — technical guides.
- `README.md` — index + PDF + versioning. Keep links in sync with files.
- `.github/workflows/docs-to-pdf.yml` — tag-triggered PDF release.

## Rules

- One concept per edit; keep FR/FS IDs stable (never renumber, only append).
- Guideline section structure is mandatory (PRD 12, DS 11, ARCH 16 sections);
  new content extends inside sections, never reshuffles them.
- ASCII diagrams only (must survive xelatex PDF). No emoji in docs.
- PDFs cover all 10 guides (`PRD` … `API-GUIDE`). `README`/`AGENTS`
  stay markdown-only (living docs).
- N/A sections must state why (single-user local tool), never silently dropped.
- ADR: single `ADR.md`, append records, never rewrite decided ones (mark
  Superseded instead).
- Validate before push:
  `for f in PRD ARCHITECTURE DESIGN-SYSTEM FSD SRS ADR SECURITY-GUIDE CODE-STYLE-GUIDE DATABASE-GUIDE API-GUIDE; do test -s "$f.md" || exit 1; done`
  plus the 10-way README `grep` (mirror the workflow).
- Release: `git tag vX.Y.Z && git push origin vX.Y.Z` → check Actions green →
  check Release has 10 readable PDFs (`test -s pdf/LANBox-*.pdf`).
- Never add Go code, binaries, or fixtures here. FSD/SRS/ADR go here later
  as separate PRs, then join the PDF list.
