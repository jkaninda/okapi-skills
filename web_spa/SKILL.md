## Okapi Web / SPA Serving

Okapi serves a front-end application (React, Vue, Svelte, Angular, …) alongside the API. Real files are served directly; any unmatched path under the prefix falls back to the index document so the client-side router takes over.

Two entry points:

```go
o.Web(prefix, dir string, cfg ...WebConfig)        // from a directory on disk
o.WebFS(prefix string, fsys fs.FS, cfg ...WebConfig) // from any fs.FS (embed.FS, os.DirFS, …)
```

> `SPA` / `SPAFS` / `SPAConfig` are **deprecated aliases** of `Web` / `WebFS` / `WebConfig`. They still work unchanged.

Register `Web`/`WebFS` **after** your API routes so the API keeps precedence.

### Serve from Disk (development)

```go
o := okapi.Default()

o.Get("/api/v1/users", listUsers)

// Serves ./web/index.html for "/", "/login", "/users/42", ...
o.Web("/", "./web")

o.Start()
```

### Serve from an Embedded FS (production, single binary)

```go
//go:embed all:web/dist
var dist embed.FS

func main() {
    o := okapi.Default()

    o.Get("/api/v1/users", listUsers)

    o.WebFS("/", dist, okapi.WebConfig{
        Root:   "web/dist", // sub-directory inside the embed.FS
        MaxAge: time.Hour,  // Cache-Control for assets
    })

    o.Start()
}
```

Use the `all:` prefix in the `//go:embed` directive so files beginning with `_` or `.` (common in build output) are included.

### WebConfig

| Field | Type | Description |
|-------|------|-------------|
| `Index` | `string` | File served for client-side routes. Defaults to `index.html`. |
| `Root` | `string` | Sub-directory inside the `fs.FS` holding the built app. **`WebFS` only** — ignored by `Web`. |
| `Exclude` | `[]string` | Extra path prefixes that must never fall back to the index. |
| `DisableAutoExclude` | `bool` | Turn off auto-excluding the top-level segment of registered routes; only `Exclude` is consulted. |
| `MaxAge` | `time.Duration` | `Cache-Control` max-age for asset files. The index is always `no-cache`. |

The zero value is valid: serve `index.html`, auto-exclude registered API routes, no asset caching header.

### Resolution Order

For every unmatched `GET`/`HEAD` request under the prefix:

1. A registered route always wins (register `Web` last).
2. If the path maps to a real file, that file is served.
3. Otherwise the index document is returned.

The top-level segment of every registered route is **auto-excluded** from the fallback, so an unknown path under an API namespace 404s instead of silently serving HTML. With `/api/v1/users` registered, `/api/v1/missing` returns `404` — not `index.html`.

### Excluding Extra Paths

```go
o.Web("/", "./web", okapi.WebConfig{
    Exclude: []string{"/metrics", "/healthz"},
})
```

### Caching Behaviour

- **Index document** — always `Cache-Control: no-cache`, so a new deploy is picked up immediately.
- **Assets** — honour `WebConfig.MaxAge`. `MaxAge: time.Hour` emits `Cache-Control: public, max-age=3600`. Zero adds no header.

### Static Files (no index fallback)

When you want plain file serving without the SPA fallback:

```go
o.Static("/assets", "public/assets")        // serve a directory, no directory listing
o.StaticFile("/favicon.ico", "favicon.ico") // serve a single file
o.StaticFS("/assets", http.FS(embedFS))     // serve from any http.FileSystem
```

Directory listing is disabled by default.

### Mounting a Standard File Server

A catch-all segment (`{any...}`) hands a whole subtree to an `http.Handler`:

```go
o.HandleHTTP("GET", "/assets/{any...}",
    http.StripPrefix("/assets/", http.FileServer(http.Dir("./public"))))
```
