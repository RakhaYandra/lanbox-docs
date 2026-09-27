# LANBox — Code Style Guide

Go-first (the binary), minimal JS (the embedded UI). Ponytail rule: no
abstraction before its third use.

## A. Go rules

- `gofmt` clean and `go vet ./...` green — non-negotiable, checked in CI.
- Naming: packages one lowercase word (`server`, `transfer`); exported
  `PascalCase` with godoc comment; unexported `camelCase`; errors `errX`;
  no stutter (`transfer.Manager`, not `transfer.TransferManager`).
- Files: one concern per file (`server.go`, `routes.go`, `middleware.go`);
  tests beside code (`path_test.go`).
- Imports grouped: stdlib, external, internal (`lanbox/internal/...`).
- Indent tabs, braces same line, double quotes, lines ≤ 100 where readable.
- Comments explain why, never narrate what; godoc on every exported symbol.

## B. React + Vite (lanbox-web repo)

- Functional components + hooks; each DESIGN-SYSTEM component is one file
  (`FileRow.jsx`, `ProgressBar.jsx`, ...). API calls live in `src/api.js`
  (`async/await` over raw promises); upload progress via XHR wrapped once.
- Error boundary at App level; per-transfer errors render inline (see
  DESIGN-SYSTEM §6 patterns). No `dangerouslySetInnerHTML` with data —
  React escapes by default.
- `const` default, never `var`; single quotes in JS/JSX; Vite dev proxy
  `/api` → `http://localhost:8080` (no hardcoded host in components).

```js
// Good
const listFiles = async (path) => {
  try {
    const res = await fetch(`/api/v1/files?path=${encodeURIComponent(path)}`, {
      headers: { Authorization: `Bearer ${token}` },
    });
    if (!res.ok) throw new Error(`List failed: ${res.status}`);
    return res.json();
  } catch (err) {
    showError('Server unreachable — check you are on the same Wi-Fi');
    throw err;
  }
};

// Bad
var listFiles = function(path) {
  return fetch('/api/v1/files?path=' + path).then((r) => r.json());
};
```

## C. Go example

```go
// Good
func (s *Server) handleList(w http.ResponseWriter, r *http.Request) {
    path, err := filesystem.Resolve(s.root, r.URL.Query().Get("path"))
    if err != nil {
        writeError(w, http.StatusBadRequest, "Invalid path")
        return
    }
    entries, err := filesystem.Browse(path)
    if err != nil {
        writeError(w, http.StatusNotFound, "Not found")
        return
    }
    writeJSON(w, entries)
}

// Bad
func (s *Server) handleList(w http.ResponseWriter, r *http.Request) {
    entries, _ := filesystem.Browse(r.URL.Query().Get("path"))
    writeJSON(w, entries)
}
```

## D. Common standards

Functions ≤ 50 lines as guidance (split on third parameter or second
nesting level, not on line count alone). Early returns over nesting.
Table-driven tests for `filesystem` and `auth`; coverage gate on those two
packages. Docs updated in the same PR as behavior.

## E. Tools & enforcement

`gofmt -l .` (empty), `go vet ./...`, `go test ./...` — all three run in
the code repo's CI. Optional local pre-commit hook running the same trio.
Formatter is `gofmt` for Go; `lanbox-web` uses oxlint (Vite default,
`react/rules-of-hooks` error, fetch-on-change exempted) with `npm run lint`
in web CI.
