## Okapi Context Utilities

`*okapi.Context` is the per-request object passed to every handler and middleware. Aliases: `okapi.C` and `okapi.Ctx` (both `= *Context`).

Related skills: `request_binding/` (binding), `response/` (writing responses), `error_handling/` (aborts), `templating/` (rendering), `sse_stream/` (SSE).

### Data Store (Thread-Safe)

```go
c.Set("key", value)
c.Get("key")          // (any, bool)
c.GetString("key")    // "" if missing or not a string
c.GetBool("key")      // false if missing or not a bool
c.GetInt("key")       // 0 if missing or not an int
c.GetInt64("key")     // 0 if missing or not an int64
c.GetTime("key")      // (time.Time, bool)
```

The store is how middleware passes values to handlers — JWT claim forwarding and `RequestID()` both write here.

### Request Inspection

```go
c.Request()            // *http.Request
c.Context()            // request context.Context
c.Path()               // raw request path, e.g. "/users/123"
c.RealIP()             // client IP, proxy-aware
c.ContentType()        // Content-Type header
c.Accept()             // []string of Accept values
c.AcceptLanguage()     // []string of Accept-Language tags (trimmed)
c.Referer()            // Referer header
c.Header("X-Key")      // one request header
c.Headers()            // map[string][]string of all request headers
c.IsWebSocketUpgrade() // Connection: Upgrade + Upgrade: websocket
c.IsSSE()              // GET with Accept: text/event-stream
c.Logger()             // *slog.Logger
c.Copy()               // shallow copy with a fresh data map — use before handing the context to a goroutine
```

### Path & Query Parameters

```go
c.PathParam("id") / c.Param("id")   // Param is the short alias
c.Query("page")                     // "" when absent
c.QueryArray("tags")                // repeated (?tags=a&tags=b) and comma-separated (?tags=a,b)
c.QueryMap()                        // first value of each query parameter
c.Params()                          // Deprecated — use PathParam
```

### Form Data & Uploads

```go
c.Form("name")                   // form value (parses the form)
c.FormValue("name")              // includes multipart form data
c.FormFile("file")               // (*multipart.FileHeader, error)
c.MaxMultipartMemory()           // current limit
c.SetMaxMultipartMemory(64 << 20) // override for this request (app default: 32 MB)
```

### Cookies

```go
c.Cookie("session")   // (string, error) — error when absent
c.SetCookie(name, value string, maxAge int, path, domain string, secure, httpOnly bool) // path defaults to "/"
```

### Headers & Status

```go
c.SetHeader("X-Request-ID", id)  // response header
c.WriteStatus(http.StatusOK)     // status only
c.Response()                     // okapi.ResponseWriter (StatusCode, BytesWritten, Flush, Hijack, Push)
c.ResponseWriter()               // underlying http.ResponseWriter
```

### Middleware Flow

```go
c.Next()                          // run the next middleware/handler in the chain
c.SetErrorHandler(handler)        // override the app error handler for this request only
```

### Test Contexts

```go
ctx, rec := okapi.NewTestContext("GET", "/books", nil) // in-memory request + httptest recorder
ctx := okapi.NewContext(o, w, r)                        // wrap an existing writer/request
```

See the `testing/` skill for assertions.

### Goroutine Safety

`*Context` is scoped to one request and is recycled once the handler returns. To use it from a goroutine that outlives the request, copy it first and capture only the values you need:

```go
o.Get("/async", func(c *okapi.Context) error {
    cp := c.Copy()
    go func() {
        cp.Logger().Info("background work", "ip", cp.RealIP())
    }()
    return c.NoContent()
})
```

For long-lived connections (WebSockets), hold state in your own struct rather than on the context.

### Configuration Note

The `*Okapi` struct has **no exported fields**. Configure it with `okapi.New(options...)`, `o.With(options...)`, or the chainable `With*` methods — never by assigning to a field such as `o.Server` or `o.Renderer`.
