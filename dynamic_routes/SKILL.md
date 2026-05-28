## Okapi Dynamic Routes

Okapi can enable/disable individual routes, whole groups, and OpenAPI documentation at runtime. Disabled routes respond with 404 — no redeploy required, and the OpenAPI document refreshes automatically.

### Disabling / Enabling a Single Route

```go
route := o.Get("/admin/console", adminHandler)

route.Disable() // requests now get 404
route.Enable()  // back online
```

### Disabling / Enabling a Whole Group

```go
admin := o.Group("/admin", jwtAuth.Middleware)
admin.Get("/users", listUsers)
admin.Get("/audit", listAudit)

admin.Disable() // every route under /admin is 404
admin.Enable()
```

### Toggling OpenAPI Documentation at Runtime

```go
o.WithOpenAPIDisabled() // /docs, /swagger, /redoc, /scalar, /openapi.* return 404

// Bring them back
o.WithOpenAPIDocs(okapi.OpenAPI{Title: "My API", Version: "1.0.0"})
```

### Marking a Route Deprecated

`Deprecated()` keeps the route active but tags it in OpenAPI so clients see a warning. Combine with `Hide()` to remove it from docs entirely.

```go
o.Get("/v1/books", legacyHandler).Deprecated()
o.Get("/internal/metrics", metricsHandler).Hide()
```

Group-level:

```go
v1 := o.Group("/api/v1").Deprecated() // all routes flagged deprecated
```

### Feature Flag Pattern

Toggle routes from config without restarting the process:

```go
type Toggles struct {
    BetaSearch bool
    Sandbox    bool
}

func registerRoutes(o *okapi.Okapi, t *Toggles) {
    search := o.Get("/search", searchHandler)
    sandbox := o.Group("/sandbox", devOnlyMiddleware)
    sandbox.Get("/echo", echoHandler)

    apply := func() {
        if t.BetaSearch { search.Enable() } else { search.Disable() }
        if t.Sandbox    { sandbox.Enable() } else { sandbox.Disable() }
    }

    apply() // initial state
    // Re-call apply() whenever t changes (config reload, admin endpoint, etc.)
}
```

### Admin Endpoint to Toggle Routes

Capture the routes you want to toggle when you register them; `o.Routes()` returns a snapshot (copy) of `Route` values, so iterating it cannot mutate the live registry.

```go
type RouteToggle struct {
    Path    string `json:"path" required:"true"`
    Enabled bool   `json:"enabled"`
}

func wireToggles(o *okapi.Okapi) {
    registry := map[string]*okapi.Route{
        "/beta/search": o.Get("/beta/search", searchHandler),
        "/sandbox":     o.Get("/sandbox", sandboxHandler),
    }

    o.Post("/admin/routes/toggle", okapi.H(func(c *okapi.Context, in *RouteToggle) error {
        route, ok := registry[in.Path]
        if !ok {
            return c.AbortNotFound("route not found")
        }
        if in.Enabled { route.Enable() } else { route.Disable() }
        return c.OK(okapi.M{"path": in.Path, "enabled": in.Enabled})
    }))
}
```

> `o.Routes()` returns a snapshot of registered routes for introspection — useful for listing/exposing what exists, not for mutating state.

### When Documentation Refreshes

The OpenAPI document is rebuilt lazily, so the next request to `/openapi.json` (or any UI route) reflects the current enable/disable state and any newly added webhooks. No cache invalidation step is needed.

### Hiding from Docs (Without Disabling)

```go
o.Get("/internal/health", healthHandler).Hide()       // still serves traffic, omitted from docs
o.Get("/internal/health", healthHandler, okapi.DocHide())
```
