## Okapi Middleware

`Middleware` and `MiddlewareFunc` are **type aliases of `HandlerFunc`** — `func(*Context) error`. Inside a middleware, call `c.Next()` to pass control down the chain. Anything before `c.Next()` runs on the way in, anything after runs on the way out.

```go
func custom(c *okapi.Context) error {
    start := time.Now()
    err := c.Next()
    log.Printf("Request took %v", time.Since(start))
    return err
}

o.Use(custom)
```

> **Signature change (v0.5.0).** Middleware is no longer `func(next HandlerFunc) HandlerFunc`. Drop the outer wrapper and replace `next(c)` with `c.Next()`:
>
> ```go
> // Before (v0.4.x)                          // After (v0.5.0+)
> func mw(next okapi.HandlerFunc) okapi.HandlerFunc {   func mw(c *okapi.Context) error {
>     return func(c *okapi.Context) error {                 err := c.Next()
>         err := next(c)                                    return err
>         return err                                    }
>     }
> }
> ```
>
> A middleware that needs configuration returns a closure with the new signature: `func RateLimit(rps int) okapi.Middleware { return func(c *okapi.Context) error { ... } }`.

### Built-in Middleware

| Middleware | Purpose |
|------------|---------|
| `okapi.LoggerMiddleware` | Structured access logging (method, URL, IP, status, duration, referer, UA). Skips WebSocket upgrades and SSE streams. Enabled in `okapi.Default()`. |
| `okapi.RequestID()` | Reads `X-Request-ID` or generates a UUID; stores in context (`"request_id"`) and echoes the header. |
| `okapi.BasicAuth{...}.Middleware` | Basic auth — constant-time compare; sends `WWW-Authenticate` on failure. |
| `okapi.JWTAuth{...}.Middleware` | JWT validation (HS256 / RS256 / JWKS), claims expression DSL, claim forwarding. |
| `okapi.BodyLimit{MaxBytes: 1<<20}.Middleware` | Rejects requests larger than `MaxBytes` with 413. |
| `okapi.Cors{...}.CORSHandler` | CORS preflight + headers (wildcards, credentials, expose headers, max-age). Usually attached via `WithCors()`. |

```go
o.Use(okapi.LoggerMiddleware)          // a plain HandlerFunc — no call parentheses
o.Use(okapi.RequestID())               // a constructor — call it
o.Use(okapi.BodyLimit{MaxBytes: 1 << 20}.Middleware)
o.Use(okapi.BasicAuth{Username: "admin", Password: "secret"}.Middleware)
o.Use(jwtAuth.Middleware)              // okapi.JWTAuth value
```

`RequestID()` reads `X-Request-ID` (or generates a UUID), stores it under the context key `request_id`, and echoes the header on the response:

```go
id := c.GetString("request_id")
```

### Attaching Middleware

```go
// Global (runs on every request)
o.Use(okapi.LoggerMiddleware, okapi.RequestID())

// Standard http.Handler middleware (gorilla, gziphandler, etc.)
o.UseMiddleware(func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("X-Version", "1.0.0")
        next.ServeHTTP(w, r)
    })
})

// Per-group
api := o.Group("/api", jwtAuth.Middleware)
api.Use(rateLimiter)
api.UseMiddleware(stdLibCompressor)

// Per-route
o.Get("/admin", adminHandler).Use(authMiddleware, auditMiddleware)

// As a RouteOption when registering routes
o.Get("/path", h, okapi.UseMiddleware(mw1, mw2))
```

### Chaining

Routes resolve middleware in this order: global → group → route. Middleware passed in earlier calls runs first.

```go
o.Use(okapi.LoggerMiddleware)
api := o.Group("/api", okapi.RequestID())
api.Get("/books", handler).Use(cacheMiddleware)

// Effective chain for GET /api/books:
//   LoggerMiddleware -> RequestID -> cacheMiddleware -> handler
```

### Standard Library Middleware Bridge

`UseMiddleware` accepts the classic `func(http.Handler) http.Handler` signature, so the whole net/http ecosystem (Gorilla handlers, gziphandler, otelhttp, etc.) plugs in directly:

```go
import "github.com/gorilla/handlers"

o.UseMiddleware(handlers.CORS(
    handlers.AllowedOrigins([]string{"*"}),
    handlers.AllowedMethods([]string{"GET", "POST"}),
))
```

### Writing a Reusable Middleware

```go
func RateLimit(rps int) okapi.Middleware {
    bucket := newBucket(rps)
    return func(c *okapi.Context) error {
        if !bucket.Allow() {
            return c.AbortTooManyRequests("rate limit exceeded")
        }
        return c.Next()
    }
}

o.Use(RateLimit(100))
api.Use(RateLimit(20)) // tighter limit on /api
```

### Aborting in Middleware

Return a non-nil error from `c.Abort*` to short-circuit the chain. Do **not** call `c.Next()` afterwards.

```go
func requireTenant(c *okapi.Context) error {
    if c.Header("X-Tenant") == "" {
        return c.AbortBadRequest("X-Tenant header is required")
    }
    return c.Next()
}
```

The response is committed once an `Abort*` helper runs, so a later write elsewhere in the chain is a silent no-op — always propagate the returned error.

### Middleware and Standard Handlers

The chain applies to `HandleStd` / `HandleHTTP` routes too, but those handlers receive `(http.ResponseWriter, *http.Request)` rather than `*Context`. Middleware written against `*Context` still runs; see the `std_compat/` skill.

### Related Skills

- Auth middleware configuration (JWT, Basic, CORS): `authentication/`
- Runtime enable/disable of routes and groups: `dynamic_routes/`
- `net/http` middleware interop: `std_compat/`
