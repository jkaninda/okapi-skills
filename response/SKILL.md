## Okapi Response & Error Handling

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
c.HTML(code, file, data)
c.HTMLView(code, templateStr, data)
c.Render(code, name, data)         // Uses configured Renderer
c.Redirect(code, location)
```

### File Serving

```go
c.ServeFile(path)
c.ServeFileFromFS(filepath, fs)
c.ServeFileAttachment(path, filename)   // Download
c.ServeFileInline(path, filename)       // Inline
```

### Structured Response (Body/Status/Header pattern)

```go
type BookResponse struct {
    Status    int    // HTTP status code
    Body      Book   // Response payload
    RequestID string `header:"X-Request-ID"`  // Response header
}
c.Respond(&response) / c.Return(&response)
```

### Response Control

```go
c.WriteStatus(code)              // Set HTTP status code
c.SetHeader(key, value)          // Set response header
c.SetCookie(name, value, maxAge, path, domain, secure, httpOnly)
c.Response()                     // Access ResponseWriter
```

### ResponseWriter Extensions

The `ResponseWriter` interface extends `http.ResponseWriter` with:

```go
StatusCode() int                          // Get written HTTP status code
BytesWritten() int                        // Get total bytes written
Close()                                   // Close the writer
Hijack() (net.Conn, *bufio.ReadWriter, error)  // Upgrade to raw TCP (WebSockets, proxies)
Flush()                                   // Flush buffered data (streaming, SSE, gzip)
Push(target string, opts *http.PushOptions) error  // HTTP/2 server push
```

### Abort Methods (use configured ErrorHandler)

Every HTTP status code has a dedicated method:

```go
c.AbortBadRequest(msg, ...err)           // 400
c.AbortUnauthorized(msg, ...err)         // 401
c.AbortForbidden(msg, ...err)            // 403
c.AbortNotFound(msg, ...err)             // 404
c.AbortConflict(msg, ...err)             // 409
c.AbortValidationError(msg, ...err)      // 422
c.AbortValidationErrors([]ValidationError, ...msg)  // 422 detailed
c.AbortTooManyRequests(msg, ...err)      // 429
c.Abort(err)                             // 500
c.AbortInternalServerError(msg, ...err)  // 500
c.AbortServiceUnavailable(msg, ...err)   // 503
// ... and all other 4xx/5xx codes
```

### Raw Error Methods (direct JSON write)

```go
c.ErrorBadRequest(message)
c.ErrorNotFound(message)
c.ErrorInternalServerError(message)
// ... etc.
```

### RFC 7807 Problem Details

```go
detail := okapi.NewProblemDetail(400, "https://example.com/errors/bad-input", "Invalid input").
    WithInstance("/books/123").
    WithExtension("field", "name").
    WithTimestamp()
c.AbortWithProblemDetail(detail)
```

### Error Handler Configuration

```go
app.WithErrorHandler(customHandler)
app.WithProblemDetailErrorHandler(&ErrorHandlerConfig{...})
app.WithSimpleProblemDetailErrorHandler()
app.WithDefaultErrorHandler()
```
