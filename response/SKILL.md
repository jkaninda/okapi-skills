## Okapi Response Helpers

For aborts, error handlers, and RFC 7807 Problem Details, see the `error_handling/` skill.

### JSON Responses

```go
c.OK(data)            // 200
c.Created(data)       // 201
c.NoContent()         // 204, empty body
c.JSON(code, data)    // custom status
```

### Other Formats

```go
c.XML(code, data)
c.YAML(code, data)
c.Text(code, data) / c.String(code, data)   // String is an alias for Text
c.Data(code, contentType, []byte)           // raw bytes with an explicit content type
c.HTML(code, file, data)                    // parse and render a template file
c.HTMLView(code, templateStr, data)         // render an inline template string
c.Render(code, name, data)                  // named template via the configured Renderer
c.Redirect(code, location)                  // writes the redirect; returns nothing
```

`c.Redirect` has no return value — write `c.Redirect(...); return nil`.

### File Serving

```go
c.ServeFile(path)
c.ServeFileFromFS(filepath, fs)         // from any http.FileSystem
c.ServeFileAttachment(path, filename)   // Content-Disposition: attachment (download)
c.ServeFileInline(path, filename)       // Content-Disposition: inline
```

### Structured Response (Body / Status / Header Pattern)

`c.Respond` (alias `c.Return`) introspects a struct and writes status, headers, cookies, and body from its fields.

```go
type BookResponse struct {
    Status    int    // HTTP status code (defaults to 200 when absent/zero)
    Body      Book   // response payload
    RequestID string `header:"X-Request-ID"` // response header
    Session   string `cookie:"session"`      // response cookie (Path=/)
}

return c.Respond(&BookResponse{Status: 201, Body: book, RequestID: rid})
```

Field rules:

| Field | Becomes |
|-------|---------|
| `Status` (int) | HTTP status code |
| `Body` | Response payload |
| tagged `header:"X-Name"` | Response header |
| tagged `cookie:"name"` | `Set-Cookie` with `Path=/` |
| **anything else** | A response header named after the field (or its `json` tag name) |

That last rule is easy to trip over: a stray field on the response struct silently becomes a header. Keep response structs to `Status`, `Body`, and explicitly tagged fields.

The output format comes from the request's `Accept` header:

| Accept contains | Written as |
|-----------------|------------|
| `application/xml` | XML |
| `application/yaml` / `text/yaml` / `application/x-yaml` | YAML |
| `application/json` | JSON |
| `text/plain` or `text/html` | Plain text (`c.String`) — **not** rendered HTML |
| anything else / absent | JSON |

`okapi.HandleIO` and `okapi.HandleO` write their typed output through the same path.

The target must be a struct or a non-nil pointer to one; otherwise Okapi responds 500.

### Response Control

```go
c.WriteStatus(code)              // write the status code only
c.SetHeader(key, value)          // set a response header
c.SetCookie(name, value, maxAge, path, domain, secure, httpOnly) // path defaults to "/"
c.Response()                     // okapi.ResponseWriter (extended interface)
c.ResponseWriter()               // the underlying http.ResponseWriter
```

### Write-Once Semantics

Once a response is committed — by any write helper, including an `Abort*` call — later calls to `c.JSON`, `c.OK`, `c.XML`, `c.Text`, `c.Render`, `c.Data`, `c.Error`, `c.AbortNotModified`, etc. are **silent no-ops** (logged at debug level). This prevents the double-body bug where a helper aborts without its return value being propagated and the caller then writes a success body:

```go
func loadBook(c *okapi.Context, id int) (*Book, error) {
    b := find(id)
    if b == nil {
        return nil, c.AbortNotFound("book not found") // response already committed
    }
    return b, nil
}

o.Get("/books/{id:int}", func(c *okapi.Context) error {
    b, err := loadBook(c, c.Param("id"))
    if err != nil {
        return err        // preferred: propagate
    }
    return c.OK(b)        // no-op if the response was already written
})
```

### ResponseWriter Extensions

`okapi.ResponseWriter` extends `http.ResponseWriter` with:

```go
StatusCode() int                                   // status written, or 0
BytesWritten() int                                 // body bytes written
Close() error                                      // close the writer if supported
Hijack() (net.Conn, *bufio.ReadWriter, error)      // raw TCP (WebSockets, proxies)
Flush()                                            // flush buffered data (streaming, SSE, gzip)
Push(target string, opts *http.PushOptions) error  // HTTP/2 server push
```

### Status Class Helpers

```go
okapi.IsError(code)        // 4xx or 5xx
okapi.IsClientError(code)  // 4xx
okapi.IsServerError(code)  // 5xx
```
