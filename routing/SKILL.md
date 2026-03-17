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
HandleStd(method, path string, h func(http.ResponseWriter, *http.Request), opts ...RouteOption)
HandleHTTP(method, path string, h http.Handler, opts ...RouteOption)
```

### Path Parameter Types

```
/books/{id}         - string parameter
/books/{id:int}     - integer parameter
/books/{id:uuid}    - UUID parameter
```

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

group.Disable() / group.Enable()       // Runtime toggle
group.WithBearerAuth()                  // Require Bearer auth
group.WithBasicAuth()                   // Require Basic auth
group.WithTags([]string{"API"})         // OpenAPI tags
group.WithSecurity(schemes)             // Security schemes
group.Deprecated()                      // Mark deprecated
group.Use(middleware)                   // Add middleware
group.UseMiddleware(stdMiddleware)      // Standard http middleware
group.Register(routes...)               // Bulk registration
group.HandleStd(method, path, handler)  // Standard http handler
group.HandleHTTP(method, path, handler) // http.Handler
```

### Route Methods

- `route.Hide()` - Hide from OpenAPI docs
- `route.Disable()` / `route.Enable()` - Runtime enable/disable
- `route.Use(middlewares...)` - Per-route middleware
- `route.WithIO(req, res)` - Set request/response schemas
- `route.WithInput(req)` - Set request schema
- `route.WithOutput(res)` - Set response schema

### Route Introspection

- `app.Routes() []Route` - List all registered routes
