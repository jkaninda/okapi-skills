## Okapi ↔ `net/http` Compatibility

Okapi implements `http.Handler` and accepts standard handlers and middleware, so a `net/http` codebase can be migrated incrementally.

### Okapi as an `http.Handler`

```go
o := okapi.Default()
o.Get("/hello", handler)

http.ListenAndServe(":8080", o)                // Okapi mounted in net/http
// or hand Okapi a pre-built server
o.StartServer(&http.Server{Addr: ":8080", ReadTimeout: 5 * time.Second})
```

### Registering Standard Handlers

Available on both `*Okapi` and `*Group`:

```go
o.HandleStd(method, path string, h func(http.ResponseWriter, *http.Request), opts ...RouteOption)
o.HandleHTTP(method, path string, h http.Handler, opts ...RouteOption)
```

```go
o.HandleStd("GET", "/greet", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello from Okapi!"))
})

type MyHandler struct{}
func (h *MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) { /* ... */ }

o.HandleHTTP("GET", "/custom", &MyHandler{})

// Whole subtree handed to a file server (catch-all segment)
o.HandleHTTP("GET", "/assets/{any...}",
    http.StripPrefix("/assets/", http.FileServer(http.Dir("./public"))))
```

Standard handlers still get the full middleware chain, routing, CORS registration, and OpenAPI entry.

### Path Parameters in a Standard Handler

Both accessors work:

```go
o.HandleStd("GET", "/users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")        // works: Okapi copies captured params onto the request
    _ = id
})

import "github.com/jkaninda/njia"

o.HandleStd("GET", "/users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := njia.Param(r, "id")      // reads the router context directly — no map allocation
    _ = id
})
```

Okapi's router captures parameters into the request context rather than through `http.ServeMux`, so it copies them onto the request with `SetPathValue` before invoking a standard handler (`HandleStd` and `HandleHTTP`, on the app and on groups alike). That bridge costs one map allocation per request and is paid only by routes that use a standard handler **and** declare parameters. `njia.Param` is the allocation-free accessor for new code; `r.PathValue` exists so an unmodified `net/http` handler keeps working.

Native Okapi handlers use `c.Param("id")` / `c.PathParam("id")`.

### Standard Middleware Bridge

`UseMiddleware` accepts the classic `func(http.Handler) http.Handler` signature on both `*Okapi` and `*Group`:

```go
o.UseMiddleware(func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("X-Powered-By", "Okapi")
        next.ServeHTTP(w, r)
    })
})

// Third-party middleware plugs in unchanged
import "github.com/gorilla/handlers"

o.UseMiddleware(handlers.CompressHandler)
api := o.Group("/api")
api.UseMiddleware(otelhttp.NewMiddleware("api"))
```

### Handler Comparison

| Aspect | `http.HandlerFunc` | `okapi.HandlerFunc` |
|--------|--------------------|---------------------|
| Signature | `func(http.ResponseWriter, *http.Request)` | `func(*okapi.Context) error` |
| Response writing | Write to `w` directly | Return `c.OK(...)`, `c.JSON(...)`, … |
| Error handling | Inline; Okapi cannot capture it | Return the error; Okapi's error handler formats it |
| Status codes | `w.WriteHeader(code)` | Helpers (`c.OK`, `c.Created`, `c.JSON`) |
| Content type | Set manually | Set by the helper |
| Path params | `r.PathValue("id")` / `njia.Param(r, "id")` | `c.Param("id")` |
| Binding & validation | Parse manually | `c.Bind(&v)` with struct-tag validation |

Standard handlers get routing and middleware but not `*okapi.Context` — so binding, validation, and the error-returning signature are unavailable inside them.

### Standard Handlers and OpenAPI

Standard handlers are ordinary routes, so they appear in the generated document and accept the same doc options:

```go
o.HandleStd("GET", "/legacy", legacyHandler,
    okapi.DocSummary("A standard handler"),
    okapi.DocTag("legacy"))
```

Okapi cannot infer request/response schemas for them — declare schemas explicitly with `okapi.DocRequestBody(...)` / `okapi.DocResponse(...)`, or use `okapi.DocHide()` to keep the route out of the spec.

### Gradual Migration

```go
o := okapi.Default()

// Phase 1: mount existing handlers untouched
o.HandleStd("GET", "/legacy/users", legacyListUsers)

// Phase 2: new endpoints use native handlers
o.Get("/api/v1/users", okapi.HandleO(func(c *okapi.Context) (*UsersOutput, error) {
    return &UsersOutput{Body: users}, nil
}))

// Phase 3: convert a legacy handler
// func legacyListUsers(w http.ResponseWriter, r *http.Request)
// becomes
// func listUsers(c *okapi.Context) error { return c.OK(users) }
```

Mixed routing is fully supported — standard and native routes coexist in one router with one middleware chain.

### Accessing the Underlying Objects

```go
o.Get("/raw", func(c *okapi.Context) error {
    r := c.Request()          // *http.Request
    w := c.Response()         // okapi.ResponseWriter (extends http.ResponseWriter)
    raw := c.ResponseWriter()  // the underlying http.ResponseWriter
    ctx := c.Context()        // request context.Context
    w.Header().Set("X-Custom", "value")
    _, _ = r, raw
    _ = ctx
    return nil
})
```

### Router Note

Since v0.10.0 the router is `github.com/jkaninda/njia` (Okapi's own router; it replaced the archived `gorilla/mux`). Routing, path variables, strict-slash redirects, and `NotFound`/`MethodNotAllowed` behave as before. The deprecated `okapi.WithMuxRouter` option now takes a `*njia.Router` and is a no-op in spirit — Okapi manages its own router and the option will be removed.
