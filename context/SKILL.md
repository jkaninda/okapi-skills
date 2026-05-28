## Okapi Context Utilities

### Data Store (Thread-Safe)

```go
c.Set("key", value)
c.Get("key")          // (any, bool)
c.GetString("key")
c.GetBool("key")
c.GetInt("key")
c.GetInt64("key")
c.GetTime("key")      // (time.Time, bool)
```

### Request Inspection

```go
c.Request()            // Underlying *http.Request
c.Context()            // Underlying context.Context
c.RealIP()             // Client IP (proxy-aware)
c.Path()               // Request path
c.ContentType()        // Content-Type header
c.Accept()             // Accept header values
c.AcceptLanguage()     // Accept-Language values
c.Referer()            // Referer header
c.IsWebSocketUpgrade() // WebSocket check
c.IsSSE()              // SSE check
c.Header("key")        // Single request header
c.Headers()            // All request headers
c.Logger()             // *slog.Logger for structured logging
c.Copy()               // Deep copy context (safe for goroutines)
```

### Path & Query Parameters

```go
c.PathParam("id") / c.Param("id")
c.Params()            // All path params (map[string]string)
c.Query("key")
c.QueryArray("key")   // Multi-value query parameter
c.QueryMap()          // All query params (map[string]string)
```

### Form Data

```go
c.Form("key")                    // Form field value
c.FormValue("key")               // Alias for Form
c.FormFile("key")                // (*multipart.FileHeader, error)
c.MaxMultipartMemory()           // Current memory limit
c.SetMaxMultipartMemory(max)     // Override limit for this request
```

### Cookies

```go
c.Cookie("name")                 // (string, error)
c.SetCookie(name, value, maxAge, path, domain, secure, httpOnly)
```

### Headers

```go
c.SetHeader(key, value)          // Set a response header
```

### Middleware Flow

```go
c.Next()              // Call next middleware/handler in chain
```

### Template Rendering

Three ways to load templates — pick the one that matches your deployment.

```go
// 1. Directory scan
tmpl, _ := okapi.NewTemplateFromDirectory("public/views", ".html", ".tmpl")
o.WithRenderer(tmpl)

// 2. Glob pattern (files)
tmpl, _ := okapi.NewTemplateFromFiles("public/views/*.html")
o.WithRenderer(tmpl)

// 3. Embedded FS (production deployments)
//go:embed views/*
var Views embed.FS
o.WithRendererFromFS(Views, "views/*.html")
```

Render in a handler:

```go
c.Render(http.StatusOK, "home", okapi.M{
    "title": "Welcome",
    "user":  user,
})
```

`okapi.M` is shorthand for `map[string]any`. Each key is accessible inside the template as `{{.keyName}}`.

### Custom Renderers

Either supply a function via `RendererFunc` or implement the `Renderer` interface on a struct (recommended in production so templates are parsed once at startup).

```go
// RendererFunc (lightweight)
o.Renderer = okapi.RendererFunc(func(w io.Writer, name string, data any, c *okapi.Context) error {
    tmpl, err := template.ParseFiles("templates/" + name + ".html")
    if err != nil { return err }
    return tmpl.ExecuteTemplate(w, name, data)
})

// Struct-based (cached templates)
type Template struct{ templates *template.Template }
func (t *Template) Render(w io.Writer, name string, data any, c *okapi.Context) error {
    return t.templates.ExecuteTemplate(w, name, data)
}
o.WithRenderer(&Template{
    templates: template.Must(template.ParseGlob("templates/*.html")),
})
```

### Static Files

```go
o.Static("/assets", "public/assets")        // Serve directory
o.StaticFile("/favicon.ico", "favicon.ico") // Serve single file
o.StaticFS("/assets", http.FS(embedFS))     // Serve from any http.FileSystem
```

Directory listing is disabled by default for security.

### TLS / HTTPS

```go
// Load TLS config from cert/key files with optional client auth
tlsConfig, err := okapi.LoadTLSConfig(certFile, keyFile, caFile, clientAuth)

// Enable TLS on the main server
o := okapi.New(okapi.WithTLS(tlsConfig))

// Run an HTTPS server alongside the HTTP server
o := okapi.Default()
o.With(okapi.WithTLSServer(":8443", tlsConfig))

// Or use net/http directly
o.Server.TLSConfig = certManager.TLSConfig() // e.g. autocert
```

Supports dual HTTP + HTTPS, client certificate authentication, and any `*tls.Config` (autocert, custom CAs, mTLS).
