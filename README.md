# LANBox Docs

[![docs](https://github.com/RakhaYandra/lanbox-docs/actions/workflows/docs-to-pdf.yml/badge.svg)](https://github.com/RakhaYandra/lanbox-docs/releases)

Official documentation for **LANBox**, a local-first LAN file sharing tool
(English). This repo is the source of requirements and design; code lives
in `lanbox` (Go API) and `lanbox-web` (React UI).

| Document | Content |
|---|---|
| [PRD.md](PRD.md) | Product Requirements — FR-xx functional + NFR ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System architecture, layers, flows ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [DESIGN-SYSTEM.md](DESIGN-SYSTEM.md) | React web UI contract — tokens, components ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [FSD.md](FSD.md) | Functional Specification — per feature, FS-xx IDs ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [SRS.md](SRS.md) | Abridged IEEE 830 — FR traceability matrix ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [ADR.md](ADR.md) | Architecture Decision Records — 8 decisions ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [SECURITY-GUIDE.md](SECURITY-GUIDE.md) | Security for a local single-user tool ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [CODE-STYLE-GUIDE.md](CODE-STYLE-GUIDE.md) | Go + minimal JS conventions ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [DATABASE-GUIDE.md](DATABASE-GUIDE.md) | No-DB state inventory + rationale ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [API-GUIDE.md](API-GUIDE.md) | Endpoint reference + conventions ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [IMPLEMENTATION.md](IMPLEMENTATION.md) | Build order, file map, formats, test gates ([PDF](https://github.com/RakhaYandra/lanbox-docs/releases/latest)) |
| [AGENTS.md](AGENTS.md) | Agent instructions for this repo (markdown only) |

## PDF

Each `v*` tag triggers the `docs-to-pdf` workflow → all 11 guides upload as
**release assets**. Download them on the
[Releases](../../releases) page. AGENTS and README stay markdown-only.

## Versioning

| Version | Date | Content |
|---|---|---|
| v0.5.0 | 2026-09-27 | Web split to lanbox-web (React+Vite, ADR-008) + PDFs (11 total) |
| v0.4.0 | 2026-09-27 | Added shares API spec + IMPLEMENTATION blueprint + PDFs (11 total) |
| v0.3.0 | 2026-09-27 | Added FSD, SRS, ADR, SECURITY/CODE/DATABASE/API guides + PDFs (10 total) |
| v0.2.0 | 2026-09-27 | Expanded per guideline (12 PRD + 11 DS + 16 ARCH sections) |
| v0.1.0 | 2026-09-27 | Initial release: PRD, ARCHITECTURE, DESIGN-SYSTEM + PDFs |

## Roadmap (docs)

Done: PRD, ARCHITECTURE, DESIGN-SYSTEM, PDF pipeline, AGENTS, FSD, SRS, ADR,
SECURITY-GUIDE, CODE-STYLE-GUIDE, DATABASE-GUIDE, API-GUIDE, IMPLEMENTATION.
Next: code repo `lanbox` implementation.
