# LANBox — Design System

Vanilla HTML/CSS/JS only (no React, no HTMX). Served embedded from the Go
binary. Mobile-first: the phone browser is the primary client.

## Tokens

- Font: system stack (`system-ui, -apple-system, sans-serif`), mono for sizes.
- Colors: `--bg: #0f1115`, `--surface: #171b22`, `--text: #e8eaf0`,
  `--muted: #9aa3b2`, `--accent: #4da3ff`, `--ok: #3ecf8e`, `--err: #ff6b6b`.
- Spacing: 4px base (`--s1..--s6`: 4/8/12/16/24/32). Radius 8px. Tap target ≥44px.

## Components

- **Header**: `LANBox` + address + transfer stats (`Active: 2 · Total: 4.8 GB`).
- **Breadcrumb**: `LANBox / Documents / Photos` — folder navigation.
- **File row**: icon (folder/file), name, size (`12 MB`, `850 MB`), checkbox
  for multi-select, tap downloads.
- **Upload button**: `[ Upload Files ]` → file picker, multi-file allowed.
- **Progress bar**: filename, bar (`████████░░ 82%`), `9.8 MB / 12 MB`, speed
  (`92 MB/s`). One bar per active transfer.
- **States**: empty (`No files yet`), error (`404 Not Found`, `401 Unauthorized`,
  `507 Insufficient Storage` — plain text, no codespeak), offline (server gone).

## Layout

Single column, max-width 720px, centered. File list scrolls; upload button
sticky bottom on mobile. No sidebar, no modal. QR block on server console,
not in web UI.

## Accessibility

Contrast ≥ 4.5:1 body text. Visible focus ring. Progress exposed via
`role="progressbar"` + text fallback. All actions keyboard-reachable.

## Contract with code

Implements exactly: `GET /api/v1/info`, `GET /api/v1/files?path=`,
`GET /api/v1/files/download?path=`, `POST /api/v1/files/upload`.
Filenames rendered escaped (no raw HTML injection). Sizes via bytes → human.
