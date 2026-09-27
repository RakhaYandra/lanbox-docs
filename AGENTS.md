# AGENTS.md — lanbox-docs

Docs-only repo. English is canonical. Source of requirements and design;
code lives in `lanbox` (Go) and `lanbox-web` (React).

## Layout

- `PRD.md` — functional + non-functional requirements (`FR-xx`, `NFR-xx`).
- `ARCHITECTURE.md` — layers, flows, dependency rule, config, security.
- `DESIGN-SYSTEM.md` — React web UI contract (tokens, components, API mapping).
- `FSD.md` — per-feature behavior (`FS-01..08` mapped to FR).
- `SRS.md` — abridged IEEE 830 + traceability matrix.
- `ADR.md` — single file, one record per decision (`ADR-001..`).
- `SECURITY-GUIDE.md`, `CODE-STYLE-GUIDE.md`, `DATABASE-GUIDE.md`
  (no-DB rationale), `API-GUIDE.md` — technical guides.
- `IMPLEMENTATION.md` — build order, file map, formats, test gates.
- `README.md` — index + PDF + versioning. Keep links in sync with files.
- `.github/workflows/docs-to-pdf.yml` — tag-triggered PDF release.

## Rules

- One concept per edit; keep FR/FS IDs stable (never renumber, only append).
- Guideline section structure is mandatory (PRD 12, DS 11, ARCH 16 sections);
  new content extends inside sections, never reshuffles them.
- ASCII diagrams only (must survive xelatex PDF). No emoji in docs.
- PDFs cover all 11 guides (`PRD` … `IMPLEMENTATION`). `README`/`AGENTS`
  stay markdown-only (living docs).
- N/A sections must state why (single-user local tool), never silently dropped.
- ADR: single `ADR.md`, append records, never rewrite decided ones (mark
  Superseded instead).
- Validate before push:
  `for f in PRD ARCHITECTURE DESIGN-SYSTEM FSD SRS ADR SECURITY-GUIDE CODE-STYLE-GUIDE DATABASE-GUIDE API-GUIDE IMPLEMENTATION; do test -s "$f.md" || exit 1; done`
  plus the 11-way README `grep` (mirror the workflow).
- Release: `git tag vX.Y.Z && git push origin vX.Y.Z` → check Actions green →
  check Release has 11 readable PDFs (`test -s pdf/LANBox-*.pdf`).
- Never add Go code, binaries, or fixtures here. New guides join the PDF
  list only via workflow + README + AGENTS update in the same PR.
