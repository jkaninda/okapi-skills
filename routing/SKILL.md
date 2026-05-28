## Okapi Routing

### HTTP Methods on `*Okapi` and `*Group`

```go
Get(path, handler, ...RouteOption) *Route
Post(path, handler, ...RouteOption) *Route
Put(path, handler, ...RouteOption) *Route
Delete(path, handler, ...RouteOption) *Route
Patch(path, handler, ...RouteOption) *Route
Head(path, handler, ...RouteOption) *Route
Options(path, handler, ...RouteOption) *Route
Any(path, handler, ...RouteOption) *Route   // All methods (Okapi only)
```

### Standard Library Handlers

```go
// On *Okapi and *Group
o.Handle(method, path, h, ...opts)                                  // generic Okapi handler
o.HandleStd(method, path, http.HandlerFunc, ...opts)                // plain http.HandlerFunc
o.HandleHTTP(method, path, http.Handler, ...opts)                   // http.Handler
```

### Path Parameter Types

```
/books/{id}         - string parameter
/books/{id:int}     - integer parameter
/books/{id:uuid}    - UUID parameter
```

Access with `c.PathParam("id")` or `c.Param("id")`. Path params declared in the path are auto-documented in OpenAPI.

### Generic Handler Wrappers

```go
// Auto-bind input struct
okapi.Handle[I](func(c *Context, input *I) error) HandlerFunc
okapi.H[I](func(c *Context, input *I) error) HandlerFunc         // Shortcut

// Bind input + return typed output
okapi.HandleIO[I, O](func(c *Context, input *I) (*O, error)) HandlerFunc

// Return typed output only
okapi.HandleO[O](func(c *Context) (*O, error)) HandlerFunc
```

### Route Groups

```go
group := app.Group("/api", middleware1, middleware2)
sub   := group.Group("/v1")

group.WithTags([]string{"API"})           // OpenAPI tags inherited by routes
group.WithTagInfo(okapi.GroupTag{...})    // Tag with description
group.WithBearerAuth() / .WithBasicAuth() // Auth scheme requirement
group.WithSecurity(schemes)               // Custom security requirements
group.Deprecated()                        // Mark group routes deprecated
group.Use(...middleware)                  // Add middleware
group.UseMiddleware(stdMiddleware)        // Standard http.Handler middleware
group.Register(routes ...RouteDefinition) // Bulk register RouteDefinition slice
group.HandleStd(method, path, handler)    // Standard http handler
group.HandleHTTP(method, path, handler)   // http.Handler
group.Okapi()                             // Back-reference to *Okapi
group.Disable() / .Enable()               // Runtime toggle (all routes 404)
```

### Route Methods

```go
route.Hide()                              // Hide from OpenAPI docs
route.Deprecated()                        // Mark deprecated in docs
route.Disable() / route.Enable()          // Runtime enable/disable (404 when disabled)
route.Use(...middlewares)                 // Per-route middleware
route.WithIO(req, res)                    // Set request/response schemas
route.WithInput(req)                      // Set request schema
route.WithOutput(res)                     // Set response schema
route.WithSecurity(...schemes)            // Per-route security requirements
```

### Static Files

```go
o.Static("/assets", "public/assets")      // Serve directory at /assets/*
o.StaticFile("/favicon.ico", "favicon.ico")
o.StaticFS("/assets", http.FS(embedFS))   // Serve from any http.FileSystem
```

Directory listing is disabled by default for security.

### Fallback Handlers

```go
o.NoRoute(func(c *okapi.Context) error { ... })  // 404 fallback
o.NoMethod(func(c *okapi.Context) error { ... }) // 405 fallback
```

### Route Introspection

```go
o.Routes() []Route                        // List all registered routes
```

### Mounting `http.Handler` & `http.HandlerFunc`

```go
o.HandleStd("GET", "/legacy", legacyHandler)

type MyHandler struct{}
func (h *MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) { ... }

o.HandleHTTP("GET", "/custom", &MyHandler{})
```

### Accessing Underlying Objects

```go
o.Get("/raw", func(c *okapi.Context) error {
    req := c.Request()   // *http.Request
    w   := c.Response()  // ResponseWriter (extends http.ResponseWriter)
    w.Header().Set("X-Custom", "value")
    return nil
})
```
