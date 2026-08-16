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
Any(path, handler, ...RouteOption) *Route   // all methods — *Okapi only
```

Generic registration: `o.Handle(method, path, handler, ...opts)`.

### Path Parameters

Both brace and colon syntax are accepted:

```
/books/{id}          /books/:id          - string parameter
/books/{id:int}      /books/:id:int      - documented as integer
/books/{id:uuid}     /books/:id:uuid     - documented as UUID
/assets/{any...}                          - catch-all segment (matches the rest of the path)
```

Read them with `c.PathParam("id")` or `c.Param("id")`. Parameters declared in the path are auto-documented in OpenAPI.

> Type hints (`:int`, `:uuid`) affect **OpenAPI schema generation only** — at runtime every parameter is a string. A name of `id` or ending in `_id` is documented as a UUID by default.

Inside a standard `net/http` handler use `r.PathValue("id")` or `njia.Param(r, "id")` — see the `std_compat/` skill.

### Standard Library Handlers

```go
o.HandleStd(method, path, func(http.ResponseWriter, *http.Request), ...opts)  // http.HandlerFunc
o.HandleHTTP(method, path, http.Handler, ...opts)                             // http.Handler

// Both also exist on *Group and inherit the group prefix + middleware
api := o.Group("/api")
api.HandleStd("GET", "/legacy", legacyHandler)
api.HandleHTTP("GET", "/custom", &MyHandler{})

// Hand a whole subtree to an http.Handler with a catch-all
o.HandleHTTP("GET", "/assets/{any...}",
    http.StripPrefix("/assets/", http.FileServer(http.Dir("./public"))))
```

### Generic Handler Wrappers

```go
okapi.Handle[I](func(c *Context, in *I) error) HandlerFunc      // bind + validate input
okapi.H[I](func(c *Context, in *I) error) HandlerFunc           // shorthand for Handle
okapi.HandleIO[I, O](func(c *Context, in *I) (*O, error)) HandlerFunc // input + typed output
okapi.HandleO[O](func(c *Context) (*O, error)) HandlerFunc      // typed output only
```

`HandleIO` / `HandleO` write the response using content negotiation from the `Accept` header (JSON by default). See the `validation/` skill for the full comparison.

### Route Groups

```go
group := o.Group("/api", middleware1, middleware2)
sub   := group.Group("/v1")            // nested; inherits the parent's middleware

group.WithTags([]string{"API"})                    // OpenAPI tags inherited by routes
group.WithTagInfo(okapi.GroupTag{                  // tag + description emitted at spec root
    Name:        "API",
    Description: "Public API",
    ExternalDocs: &okapi.ExternalDocs{URL: "https://docs.example.com"},
})
group.WithBearerAuth() / group.WithBasicAuth()     // auth scheme requirement in docs
group.WithSecurity([]map[string][]string{{"bearerAuth": {}}})
group.Deprecated()                                 // mark all routes deprecated
group.Disable() / group.Enable()                   // runtime toggle (all routes 404)
group.Use(...middleware)                           // add Okapi middleware
group.UseMiddleware(stdMiddleware)                 // add std-lib middleware
group.Register(routes ...RouteDefinition)          // bulk register
group.Okapi() *Okapi                               // back-reference
```

`o.Group("")` panics — a group needs a non-empty prefix.

Standalone construction: `okapi.NewGroup("/api", o, middlewares...)`.

### Route Methods

```go
route.Hide() *Route                       // hide from OpenAPI docs (still serves traffic)
route.Deprecated() *Route                 // mark deprecated in docs
route.Disable() / route.Enable() *Route   // runtime enable/disable (404 when disabled)
route.Use(...Middleware)                  // per-route middleware
route.WithInput(req any) *Route           // request schema (bind + validate + document)
route.WithOutput(res any) *Route          // response schema
route.WithIO(req, res any) *Route         // both (either may be nil)
route.WithSecurity(...map[string][]string) *Route
```

`Route` exposes `Name`, `Path`, and `Method` fields.

### Fallback Handlers

```go
o.NoRoute(func(c *okapi.Context) error {  // 404 — path not matched
    return c.AbortNotFound("Custom 404 - Not found")
})

o.NoMethod(func(c *okapi.Context) error { // 405 — path matched, method not allowed
    return c.AbortMethodNotAllowed("Custom 405 - Method Not Allowed")
})
```

### Trailing Slashes

```go
o := okapi.New(okapi.WithStrictSlash(true)) // redirect /books/ -> /books
```

### Static Files & SPA

```go
o.Static("/assets", "public/assets")
o.StaticFile("/favicon.ico", "favicon.ico")
o.StaticFS("/assets", http.FS(embedFS))
o.Web("/", "./web")            // SPA with index fallback — register AFTER API routes
o.WebFS("/", dist, okapi.WebConfig{Root: "web/dist"})
```

Directory listing is disabled by default. Details in the `web_spa/` skill.

### Route Introspection

```go
for _, r := range o.Routes() { // snapshot of []Route — mutating it does not affect the registry
    fmt.Println(r.Method, r.Path, r.Name)
}
```

### Accessing Underlying Objects

```go
o.Get("/raw", func(c *okapi.Context) error {
    req := c.Request()   // *http.Request
    w   := c.Response()  // okapi.ResponseWriter (extends http.ResponseWriter)
    w.Header().Set("X-Custom", "value")
    return nil
})
```
