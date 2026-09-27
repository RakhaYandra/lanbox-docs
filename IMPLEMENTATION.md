# LANBox — Implementation Blueprint

Build order, file map, formats, and test gates for the code repos:
`lanbox` (`github.com/RakhaYandra/lanbox`, Go 1.27) and `lanbox-web`
(React + Vite). Normative behavior lives in
FSD/API-GUIDE; this file says what to build first and how to prove it.

## 1. Repo blueprint

```
lanbox/
├── cmd/lanbox/main.go            # cobra root: serve, status, send, receive, stop, version
├── internal/
│   ├── server/{server.go,routes.go,middleware.go}   # ServeMux, auth, semaphore, slog, --web-dir static
│   ├── transfer/{upload.go,download.go,progress.go,limit.go,checksum.go}
│   ├── filesystem/{browser.go,path.go}              # gate: Clean -> Resolve -> Validate
│   ├── discovery/{network.go,qr.go}                 # LAN IP print, QR render
│   ├── auth/{token.go,pin.go}                       # rand 32B, constant-time compare
│   └── config/{config.go}                           # flags > env > file > defaults
├── Makefile  go.mod  README.md  .gitignore

lanbox-web/                        # separate repo, React + Vite
├── src/{App.jsx,api.js,components/{Header,Breadcrumb,FileRow,UploadButton,ProgressBar,Icon}.jsx}
├── index.html  package.json  vite.config.js  # dev proxy /api -> :8080
└── dist/                          # build output, served by lanbox --web-dir (gitignored)
```

Import rule (ARCHITECTURE): `server -> transfer -> filesystem -> stdlib`.
Inner layers never import `server` or `cobra`. `filesystem` and `auth`
stay stdlib-only.

## 2. Config format

File `~/.config/lanbox/config.toml`, parsed manually (no Viper):

```toml
dir = "~/LANBox"
port = 8080
host = "0.0.0.0"
pin_enabled = true
limit_mbps = 0  # 0 = unlimited
```

Precedence: CLI flags, then `LANBOX_DIR/PORT/HOST/PIN/LIMIT`, then this
file, then defaults.

## 3. PID format

File `~/.local/share/lanbox/lanbox.pid`, single-line JSON:

```json
{"pid": 18293, "address": "192.168.1.10:8080", "dir": "/home/user/LANBox"}
```

Written at boot, removed on clean shutdown. `status` reports `not running`
and offers cleanup when the PID is dead.

## 4. CLI reference

| Command | Flags / env | Exit codes |
|---|---|---|
| `serve` | `--dir --port --host --limit`, `LANBOX_*` | 0 ok, 1 bad dir/port |
| `status` | — (reads PID file) | 0 running, 2 not running |
| `send <file>` | `--to <addr>`, `LANBOX_TOKEN` | 0 ok, 1 no file/no connect |
| `receive <url>` | `--out <dir>`, `LANBOX_TOKEN` | 0 ok, 1 no connect |
| `stop` | — (kills PID) | 0 ok, 2 not running |
| `version` | — | 0 |

## 5. Tasks per milestone

M1 core + web (DoD: browser-to-server upload/download roundtrip of 1 GB):
1. `config` + cobra skeleton + `version`. 2. `filesystem/path` gate +
table tests. 3. `server` + routes info/list. 4. download streaming. 5.
upload streaming + progress. 6. `web` UI (list, upload, bar). 7. graceful
shutdown + slog. 8. README install + usage.

M2 CLI + security (DoD: send/receive works, 401/400 matrix green):
1. token/PIN middleware. 2. `send` client + progress. 3. `receive` client.
4. `status`/`stop` via PID file. 5. QR render in serve output. 6. traversal
+ no-token test sweeps.

M3 reliability + V2 (DoD: 4 concurrent, resume, checksum, share expiry,
limit): 1. semaphore 4 + 429 queue. 2. disconnect cancel + partial
cleanup. 3. `Range` resume. 4. SHA-256 verify both sides. 5. shares
endpoints + expiry sweep. 6. `--limit` token bucket. 7. 2/4/8-client load
run + benchmark notes.

## 6. Test plan

- Unit: `go test ./internal/filesystem ./internal/auth` (table-driven,
  coverage gate on these two packages).
- Integration: boot serve → upload 100 MB → download → `sha256sum` equal
  on both sides.
- Concurrency: 2/4/8 parallel `curl` uploads; 5th must 429 with
  `Retry-After`.
- Security: 5-pattern traversal matrix (all 400), endpoint sweep without
  token (all 401), malicious upload names.
- Fixtures: `dd if=/dev/urandom of=fixture-100M bs=1M count=100` (1 GB:
  `count=1024`; 5 GB optional), generated locally, never committed.
- Gates: `gofmt -l .` empty, `go vet ./...` green, full `go test ./...`
  green.

## 7. Makefile / CI (code repo)

```make
build: go build -o bin/lanbox ./cmd/lanbox        # GOOS=linux|darwin|windows
test:  go test ./...
vet:   go vet ./...
fmt:   gofmt -l .
```

CI (code repo, separate from docs-PDF CI): run `fmt` + `vet` + `test` +
the traversal matrix on every push. `lanbox-web` CI: `npm run lint` +
`npm run build` + a smoke check that the 4 endpoints exist in `src/api.js`.
