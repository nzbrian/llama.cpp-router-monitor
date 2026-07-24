# Multi-Instance Monitoring Plan

## Goal

Monitor several llama.cpp instances at once on a single page, with each instance
proxied through its own dedicated port and differentiated in the GUI.

## Architecture

```
                    ┌─────────────────────┐
                    │   Server (shared)   │
                    │   DB + EventHub     │
                    └────────┬────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        :9090 (monitor)  :9091 (proxy→A)  :9092 (proxy→B)
     monitorHandler    proxyHandler(A)   proxyHandler(B)
```

- **One shared `Server`** — single SQLite DB, single EventHub, single HTTP client.
- **`monitorHandler`** — serves all `/_monitor/*` endpoints (UI, API, SSE). No proxying.
- **`proxyHandler{backend: "http://a:8080"}`** — pure proxy. Each listener is hardcoded to one backend via context injection.

## Configuration

| Env Var | Description | Example |
|---|---|---|
| `MONITOR_LISTEN_ADDR` | Management/UI port (serves `/_monitor/*` only) | `:9090` |
| `BACKEND_URLS` | Comma-separated llama.cpp backend URLs | `http://a:8080,http://b:8080` |
| `LISTEN_ADDRS` | Comma-separated proxy ports (1:1 with backends) | `:9091,:9092` |

Other env vars remain unchanged (`ALLOW_DYNAMIC_BACKEND`, `DATA_DIR`, etc.).

## Backend Changes (`main.go`)

### Config struct
- Replace `ListenAddr`/`DefaultBackend` with `MonitorListenAddr string` + `Backends []BackendConfig{URL, ListenAddr}`.

### loadConfig()
- Parse `BACKEND_URLS` + `LISTEN_ADDRS` as parallel comma-separated lists.
- Validate equal length (fatal if mismatch).
- Validate each backend URL with existing `validateBackendURL`.
- Fatal if `BACKEND_URLS` is empty.

### Handler types
- **`proxyHandler`**: wraps `*Server` + `backend string`. Injects backend into request context, delegates to `handleProxy`.
- **`monitorHandler`**: wraps `*Server`. Delegates to `handleMonitor` for all requests (no path branching).

### selectBackend()
- Check `r.Context().Value(backendOverrideKey)` first (set by proxyHandler).
- Falls back to `cfg.Backends[0].URL`.
- Dynamic overrides (`X-Backend-URL`, `?backend=`) still work if `ALLOW_DYNAMIC_BACKEND=true`.

### pollBackendMetrics()
- Loop over `s.cfg.Backends`, scrape `/metrics` from each.
- Rename existing function body to `pollSingleBackend(ctx, url)`.

### getStats()
- Add `by_backend []map[string]any` — separate `GROUP BY backend_url` query returning per-backend stats.
- Add `Backend string` field to `RequestFilter`.
- Wire backend filter into `appendRequestFilterSQL`.

### New endpoint: `/_monitor/backends`
- Returns `{items: [{url, listen_addr}]}` from `s.cfg.Backends`.

### main()
- Start 1 goroutine for monitor listener + N goroutines for proxy listeners.
- Duplicate port check at startup.
- `select{}` to block forever.

## Frontend Changes

### HTML (`web/index.html`)
- New `<section id="backendBreakdown">` between summary grid and workspace.
- New "Backend" column in request table header + body.
- New `<select id="fBackend">` in filter panel.

### JavaScript (`web/app.js`)
- `renderBackendBreakdown(stats)` — renders per-backend compact cards from `stats.by_backend`.
- Populate `#fBackend` dropdown options dynamically from stats.
- Add backend column rendering in `renderRequests()`.
- `collectFilters()` / `resetFilters()` — include/exclude backend filter.

### CSS (`web/styles.css`)
- `.backend-breakdown` section styles (horizontal scroll strip).
- `.backend-card` compact card styling.
- Backend column width + text truncation.

## Test Changes (`main_test.go`)

- **`newTestServer`** signature: return `(*Server, http.Handler, *sql.DB, func())`.
- Update existing proxy tests to use returned handler.
- **New tests**: config parsing, by_backend stats, backend filter, multi-backend routing.

## Docker (`docker-compose.yml`)

```yaml
ports:
  - "9090:9090"   # monitor UI + API
  - "9091:9091"   # proxy → backend 1
  - "9092:9092"   # proxy → backend 2
environment:
  - MONITOR_LISTEN_ADDR=:9090
  - BACKEND_URLS=${BACKEND_URLS:-http://host.docker.internal:8080}
  - LISTEN_ADDRS=${LISTEN_ADDRS:-:9091}
```

## Implementation Order

1. Config struct + `loadConfig()` + validation
2. Handler types (`proxyHandler`, `monitorHandler`) + context injection
3. `selectBackend()` context check + `main()` multi-listener startup
4. `pollBackendMetrics()` iteration over backends
5. Stats: `by_backend` breakdown + backend filter in SQL
6. New `/_monitor/backends` endpoint
7. Frontend: HTML → JS → CSS
8. Tests: update existing + add new
9. Docker compose updates
10. Run `go test` + build to verify
