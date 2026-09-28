# LANBox — Code Style Guide

Go-first (the binary), strict TypeScript (the `lanbox-web` repo). Ponytail rule: no
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

## B. React + Vite + TypeScript (lanbox-web repo)

- Strict TS (`tsc --noEmit` green in CI): typed props per component
  (`FileRow.tsx`, `ProgressBar.tsx`, ...), shared types in `src/api.ts`
  (`Entry`, `TransferState`, `UploadRecord`); no explicit `any`
  (grep gate), `unknown` + narrowing in catch blocks (`HttpError` class).
- Functional components + hooks; API calls live in `src/api.ts`
  (`async/await` over raw promises); upload progress via XHR wrapped once.
- Error boundary at App level; per-transfer errors render inline (see
  DESIGN-SYSTEM §6 patterns). No `dangerouslySetInnerHTML` with data —
  React escapes by default.
- `const` default, never `var`; single quotes in TS/TSX; Vite dev proxy
  `/api` → `https://localhost:8080` (`secure: false` dev only).

```ts
// Good
const listFiles = async (path: string): Promise<FileList> => {
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
var listFiles = function(path: string) {
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
