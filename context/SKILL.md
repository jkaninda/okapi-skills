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
c.Header("key")        // Single header
c.Headers()            // All headers
c.Logger()             // *slog.Logger for structured logging
c.Copy()               // Deep copy context (safe for goroutines)
```

### Path & Query Parameters

```go
c.PathParam("id") / c.Param("id")
c.Params()            // All path params
c.Query("key")
c.QueryArray("key")
c.QueryMap()
```

### Form Data

```go
c.Form("key")                    // Form field value
c.FormValue("key")               // Alias for Form
c.FormFile("key")                // (*multipart.FileHeader, error)
```

### Cookies

```go
c.Cookie("name")
c.SetCookie(name, value, maxAge, path, domain, secure, httpOnly)
```

### Middleware Flow

```go
c.Next()              // Call next middleware/handler in chain
```

### Template Rendering

```go
// From directory
tmpl, _ := okapi.NewTemplateFromDirectory("views", ".html")
app.WithRenderer(tmpl)

// From embedded FS
//go:embed views/*
var Views embed.FS
tmpl, _ := okapi.NewTemplate(Views, "views/*.html")
app.WithRenderer(tmpl)

// Render in handler
c.Render(http.StatusOK, "home", okapi.M{"title": "Welcome"})
c.HTML(http.StatusOK, "home.html", data)
```

### Static Files

```go
app.Static("/assets", "public/assets")       // Serve directory
app.StaticFile("/favicon.ico", "favicon.ico") // Serve single file
app.StaticFS("/assets", http.FS(embedFS))     // Serve from http.FileSystem
```

Directory listing is disabled by default for security.

### TLS / HTTPS

```go
// Load TLS config from cert/key files with optional client auth
tlsConfig, err := okapi.LoadTLSConfig(certFile, keyFile, caFile, clientAuth)

// Enable TLS on the main server
app.WithTLS(tlsConfig)

// Run a separate TLS server alongside HTTP
app.WithTLSServer(":443", tlsConfig)
```

Supports dual protocol (HTTP + HTTPS simultaneously), client certificate authentication, and custom `*tls.Config`.
