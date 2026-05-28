## Okapi Response Helpers

For error handling (Abort methods, RFC 7807 Problem Details, custom error handlers), see the `error_handling/` skill.

### JSON Responses

```go
c.OK(data)            // 200
c.Created(data)       // 201
c.NoContent()         // 204
c.JSON(code, data)    // Custom status
```

### Other Formats

```go
c.XML(code, data)
c.YAML(code, data)
c.Text(code, data) / c.String(code, data)
c.Data(code, contentType, []byte)
c.HTML(code, file, data)            // Render a file directly
c.HTMLView(code, templateStr, data) // Render an inline template string
c.Render(code, name, data)          // Render a named template via the configured Renderer
c.Redirect(code, location)
```

### File Serving

```go
c.ServeFile(path)
c.ServeFileFromFS(filepath, fs)         // From any http.FileSystem
c.ServeFileAttachment(path, filename)   // Content-Disposition: attachment (download)
c.ServeFileInline(path, filename)       // Content-Disposition: inline
```

### Structured Response (Body/Status/Header Pattern)

`c.Respond` / `c.Return` introspect a struct and write the body, status, and headers from its fields. Fields tagged `header:` become response headers; the `Body` field becomes the payload; `Status` (int) selects the HTTP status.

```go
type BookResponse struct {
    Status    int               // HTTP status code (defaults to 200)
    Body      Book              // Response payload
    RequestID string            `header:"X-Request-ID"`  // Response header
    SetCookie string            `cookie:"session"`       // Response cookie
}

c.Respond(&BookResponse{Status: 200, Body: book, RequestID: rid})
c.Return(&BookResponse{Body: book})
```

The output format is chosen from the request's `Accept` header (JSON / XML / YAML / plain). Defaults to JSON.

### Response Control

```go
c.WriteStatus(code)              // Set HTTP status code (no body)
c.SetHeader(key, value)          // Set response header
c.SetCookie(name, value, maxAge, path, domain, secure, httpOnly)
c.Response()                     // Access ResponseWriter (extended interface)
c.ResponseWriter()               // Access underlying http.ResponseWriter
```

### ResponseWriter Extensions

The `ResponseWriter` interface extends `http.ResponseWriter` with:

```go
StatusCode() int                          // Get written HTTP status code
BytesWritten() int                        // Get total bytes written
Close() error                             // Close the writer
Hijack() (net.Conn, *bufio.ReadWriter, error)  // Raw TCP (WebSockets, proxies)
Flush()                                   // Flush buffered data (streaming, SSE, gzip)
Push(target string, opts *http.PushOptions) error  // HTTP/2 server push
```
