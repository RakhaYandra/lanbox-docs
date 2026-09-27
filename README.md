# LANBox Docs

[![docs](https://github.com/RakhaYandra/lanbox-docs/actions/workflows/docs-to-pdf.yml/badge.svg)](https://github.com/RakhaYandra/lanbox-docs/releases)

Official documentation for **LANBox**, a local-first LAN file sharing tool
(English). This repo is the source of requirements and design; the Go
implementation lives in the future `lanbox` repo.

| Document | Content |
|---|---|
| [PRD.md](PRD.md) | Product Requirements — FR-xx functional + NFR ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System architecture, layers, flows ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [DESIGN-SYSTEM.md](DESIGN-SYSTEM.md) | Vanilla web UI contract — tokens, components ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [AGENTS.md](AGENTS.md) | Agent instructions for this repo (markdown only) |

## PDF

Each `v*` tag triggers the `docs-to-pdf` workflow → PRD, ARCHITECTURE,
DESIGN-SYSTEM PDFs upload as **release assets**. Download them on the
[Releases](../../releases) page. AGENTS and README stay markdown-only.

## Versioning

| Version | Date | Content |
|---|---|---|
| v0.1.0 | 2026-09-27 | Initial release: PRD, ARCHITECTURE, DESIGN-SYSTEM + PDFs |

## Roadmap (docs)

Done: PRD, ARCHITECTURE, DESIGN-SYSTEM, PDF pipeline, AGENTS.
Next: FSD (endpoint spec), SRS (IEEE 830 traceability), ADRs.
