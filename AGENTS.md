# AGENTS.md — llama.cpp Router Monitor

## Repo structure

- **`main.go`** (~1880 lines) — entire backend. Single package, single file. No other Go packages exist.
- **`web/app.js`** — vanilla JS SPA (no framework). Served by `handleUI` from the same binary.
- **`web/index.html`**, **`web/styles.css`**, **`web/favicon.svg`** — frontend assets.
- **`main_test.go`** — all tests live alongside `main.go`.

## Run / build

```bash
# run tests
go test ./... -count=1

# build (matches Dockerfile exactly)
CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o llama-cpp-router-monitor .

# Docker
docker compose up -d --build
```

## Architecture facts an agent will miss

- **Everything is in `main.go`.** There are no sub-packages, no `cmd/`, no `internal/`. Adding a new file in package `main` is fine; adding a new Go package is not the convention.
- **Routing split:** any path starting with `/_monitor` goes to `handleMonitor`; everything else is proxied (`handleProxy`). The proxy never inspects paths — it blindly forwards to the backend.
- **Data on disk:** `DATA_DIR/monitor.db` (SQLite) and `DATA_DIR/raw/YYYY-MM-DD/{id}-{request|response}.gz`. The gzip files are the source of truth for raw payloads; the DB only stores metadata + file paths.
- **Repair loop runs at startup** (`repairStuckRequests`). It scans for rows with `status_code=0` and no `response_raw_path`, reads the gzipped response from disk, re-parses tokens/cache/timings, and updates the row. If you add a new field to `RequestRecord`, remember to backfill it in this function too — otherwise stale data persists until cleanup.
- **SQLite migrations are inline** via `PRAGMA table_info` + `ALTER TABLE ADD COLUMN` (`ensureRequestColumn`). New columns on `requests` must be added here, not in a migration file.
- **SSE event format:** `event: request\ndata: {json}\n\n`. The frontend uses `EventSource` and listens for the `"request"` custom event type (not generic messages). Keep this exact format when broadcasting new events.

## Testing quirks

- Tests use `httptest.NewServer` for both the monitor (`svc`) and a fake backend.
- `newTestServer(t, backendURL)` creates an in-memory SQLite DB in `t.TempDir()`. The cleanup function removes `.db-shm`/`.db-wal` files and closes the DB — call it.
- Streaming tests coordinate via channels (`backendStarted`, `releaseBackend`) to observe in-flight state. Don't add `time.Sleep` where a channel would work.
- `seedRequest` calls both `insertRequest` + `finishRequest` — useful for manually inserting records without going through the proxy.

## Constraints worth knowing

- **No CGO.** The build flag `CGO_ENABLED=0` is required (matches Dockerfile). Adding a dependency that requires CGO will break the build.
- **Dynamic backend override:** per-request routing via `X-Backend-URL` header or `?backend=` query param. Validated by `validateBackendURL` (must start with `http://` or `https://`).
- **Path traversal guard:** `deleteRequestByID` resolves the raw file path relative to `DATA_DIR` and refuses deletion if it escapes (`filepath.Rel` check). Preserve this when touching delete logic.
- **Cleanup runs hourly.** Raw payload directories older than `RETENTION_DAYS` are removed via `os.RemoveAll`. Setting `RETENTION_DAYS=0` disables cleanup entirely.
- **Token parsing falls back to llama.cpp legacy fields:** if `usage` is absent, the parser checks `tokens_evaluated`, `tokens_predicted`, `tokens_evaluated_ms`, `tokens_predicted_ms`. New fields should also be checked here (`parseJSONResponseMeta`).

## Frontend conventions

- All UI state lives in a single `state` object at the top of `app.js`.
- Filter persistence uses `localStorage` keys `llama-cpp-router-monitor.filters.collapsed` and `llama-cpp-router-monitor.metrics.mode`.
- Auto-refresh intervals: stats every 5s, request list every 9s, model options every 30s. Event-driven refresh is debounced to 180ms.
- The structured response view (`buildStructuredResponseView`) parses SSE chunks into a JSON summary — useful when adding new streaming payload shapes.

## Adding a new monitor endpoint

1. Add a case in the `switch` inside `handleMonitor`.
2. If it queries the DB, prefer parameterized queries (see `appendRequestFilterSQL` for the pattern).
3. If it writes to the DB, wrap in `retryDBWrite` to handle SQLite locking.
4. If it affects observable state, broadcast via `s.hub.Broadcast`.
