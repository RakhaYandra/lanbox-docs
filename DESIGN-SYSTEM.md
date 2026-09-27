# LANBox — Design System

Vanilla HTML/CSS/JS only (no React, no HTMX). Served embedded from the Go
binary. Mobile-first: the phone browser is the primary client.

## 1. Color Palette

| Role | Hex | Usage |
|---|---|---|
| `bg` | `#0F1115` | Page background |
| `surface` | `#171B22` | File rows, cards |
| `track` | `#242B36` | Progress track |
| `text` | `#E8EAF0` | Body, filenames |
| `muted` | `#9AA3B2` | Sizes, meta, stats |
| `accent` | `#4DA3FF` | Links, primary button, progress fill, focus |
| `ok` | `#3ECF8E` | Verified, completed |
| `err` | `#FF6B6B` | Errors, failed bars |

## 2. Typography

System stack for text (`system-ui, -apple-system, sans-serif`), monospace
(`ui-monospace, monospace`) for sizes, speeds, hashes.

| Level | Size/Weight/Line | Usage |
|---|---|---|
| H1 | 20 / 700 / 28 | Header `LANBox` |
| H2 | 16 / 600 / 24 | Section `Files` |
| Body | 14 / 400 / 20 | Filenames, buttons |
| Caption | 12 / 400 / 16 | Sizes, speed, stats |

## 3. Spacing / Grid

4px base: `--s1..s6 = 4/8/12/16/24/32`. Row padding 12 vertical, 16
horizontal; list gap 4; radius 8. Single column, max-width 720px, centered.
Upload button sticky at bottom on mobile.

## 4. Icons

Inline SVG, no library (fits the single binary). Stroke 1.5px, 20px box,
`currentColor`. Set: folder, file, upload-arrow, download-arrow, check, x,
chevron-right (breadcrumb). New icons go in one `icons.js` as
`icon-<name>`; never emoji in UI (emoji breaks the xelatex PDF build).

## 5. Components

### Header

Anatomy: `LANBox | <address> | Active N, Total X GB`. States: default,
offline (muted + `Server unreachable`). Usage: always visible, never scrolls
away on desktop.

### Breadcrumb

Anatomy: `LANBox / Documents / Photos` with chevron separators; last segment
is plain text. States: link (accent on hover), current (text), focus-visible
ring. Usage: one tap per level up.

### File row

Anatomy: `[icon 20px] [name, flex] [size mono] [checkbox]`. States:
default, hover (lighter surface), active (pressed), selected (accent left
border), focus-visible ring, disabled at 50% when offline. Variations: dir
(folder icon, tap navigates) vs file (file icon, tap downloads). Usage: tap
row acts; checkbox only selects for multi-transfer.

### Upload button

Anatomy: `[ Upload Files ]`, primary accent, full-width on mobile, sticky
bottom. States: default, hover (brighten), active, disabled (4 transfers
busy, muted + `Server busy`), focus-visible ring. Usage: opens multi-file
picker.

### Progress bar

Anatomy: `[name] [bar] [82%] [9.8 / 12 MB, 92 MB/s]`, one per transfer.
States: running (accent), done (green + `Verified` on V2 checksum),
error (red + message + retry button). Usage: text always mirrors the bar
(screen-reader safe).

### QR console block

Server terminal only, not web UI: LAN URL, PIN, and QR together so a phone
camera connects in one scan.

## 6. Patterns

- Multi-upload queue: 4 run, rest show `Queued`; excess gets 429 and the UI
  shows `Queued — server busy, retrying`.
- Loading: bar + percent + bytes + speed, updates at most 5 Hz.
- Errors (plain words, never codespeak): `Wrong PIN — try again` (401),
  `Not found` (404), `Disk full` (507), `Invalid path` (400).
- Empty: `No files yet — upload to begin`.
- Offline: `Server unreachable — check you are on the same Wi-Fi`.

## 7. Accessibility Guidelines

WCAG AA: body contrast about 13.5:1, muted about 5.1:1 (both above 4.5:1).
Visible 2px accent focus ring. Progress uses `role="progressbar"` with
`aria-valuenow` plus text fallback. Full keyboard reach (breadcrumb, list,
upload). Touch targets at least 44px.

## 8. Animation / Motion

Progress repaints via `requestAnimationFrame`, throttled to 200ms. No layout
animations. `prefers-reduced-motion` switches bars to discrete jumps with no
transitions.

## 9. Responsive Breakpoints

| Width | Layout |
|---|---|
| 360px phone (primary) | Full-width list, sticky upload button |
| 720px | Max content width, centered |
| 1024px and up | Content stays 720px centered, never stretches |

## 10. Brand Voice & Tone

Short, technical, friendly. Examples: `Serving ~/LANBox`, `Resuming from
820 MB...`, `Transfer complete — 1.4 GB in 18.2s`, `Wrong PIN — try again`.
Forbidden: error codespeak (`ERR_XFER_04`), hype words (`instant`), blame.

## 11. Contract with code

Implements exactly: `GET /api/v1/info`, `GET /api/v1/files?path=`,
`GET /api/v1/files/download?path=`, `POST /api/v1/files/upload`.
Filenames rendered escaped (no raw HTML). Sizes formatted bytes to human
(B/KB/MB/GB, base 1024). Upload limits surface as 507 with the disk-full
message.
