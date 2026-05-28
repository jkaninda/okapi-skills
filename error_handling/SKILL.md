## Okapi Error Handling

Okapi has a flexible error handling system. Handlers return `error` from `c.Abort*` helpers and the configured `ErrorHandler` writes the response. Three response formats are built in: default JSON envelope, fully custom, and RFC 7807 Problem Details.

### Aborting from a Handler

```go
o.Post("/books", func(c *okapi.Context) error {
    book := &Book{}
    if err := c.Bind(book); err != nil {
        return c.AbortBadRequest("Invalid request body", err)
    }
    return c.Created(book)
})
```

Default response:

```json
{
  "code": 400,
  "message": "Invalid request body",
  "details": "field Name is required",
  "timestamp": "2026-02-09T21:34:17+01:00"
}
```

### Abort Methods (Routed Through the Configured Error Handler)

| Method | Status |
|--------|--------|
| `c.AbortBadRequest(msg, ...err)` | 400 |
| `c.AbortUnauthorized(msg, ...err)` | 401 |
| `c.AbortForbidden(msg, ...err)` | 403 |
| `c.AbortNotFound(msg, ...err)` | 404 |
| `c.AbortMethodNotAllowed(msg, ...err)` | 405 |
| `c.AbortConflict(msg, ...err)` | 409 |
| `c.AbortGone(msg, ...err)` | 410 |
| `c.AbortPreconditionFailed(msg, ...err)` | 412 |
| `c.AbortRequestEntityTooLarge(msg, ...err)` | 413 |
| `c.AbortUnsupportedMediaType(msg, ...err)` | 415 |
| `c.AbortValidationError(msg, ...err)` | 422 |
| `c.AbortValidationErrors([]ValidationError, ...msg)` | 422 with structured field errors |
| `c.AbortTooManyRequests(msg, ...err)` | 429 |
| `c.AbortInternalServerError(msg, ...err)` / `c.Abort(err)` | 500 |
| `c.AbortNotImplemented(msg, ...err)` | 501 |
| `c.AbortBadGateway(msg, ...err)` | 502 |
| `c.AbortServiceUnavailable(msg, ...err)` | 503 |
| `c.AbortGatewayTimeout(msg, ...err)` | 504 |

For any other status:

```go
return c.AbortWithError(http.StatusTeapot, err)
```

### Direct Error Writes (Bypass the Error Handler)

Use these when you want a JSON error written verbatim without going through the configured `ErrorHandler`:

```go
c.ErrorBadRequest(message)
c.ErrorUnauthorized(message)
c.ErrorForbidden(message)
c.ErrorNotFound(message)
c.ErrorInternalServerError(message)
// ... etc.
```

### Custom Error Handler

Replace the default response format entirely:

```go
o.WithErrorHandler(func(c *okapi.Context, code int, message string, err error) error {
    return c.JSON(code, map[string]any{
        "success": false,
        "error": map[string]any{
            "code":    code,
            "message": message,
            "details": err.Error(),
        },
    })
})
```

Reset back to the default handler:

```go
o.WithDefaultErrorHandler()
```

### RFC 7807 Problem Details

```go
// Simplest: default fields, Content-Type: application/problem+json
o := okapi.Default()
o.WithSimpleProblemDetailErrorHandler()
```

Response:

```json
{
  "type": "about:blank",
  "title": "Bad Request",
  "status": 400,
  "detail": "field Name is required",
  "instance": "/books"
}
```

Full configuration:

```go
o.WithProblemDetailErrorHandler(&okapi.ErrorHandlerConfig{
    Format:           okapi.ErrorFormatProblemJSON, // or ErrorFormatProblemXML
    TypePrefix:       "https://api.example.com/errors/",
    IncludeInstance:  true,
    IncludeTimestamp: true,
    CustomFields: map[string]any{
        "api_version": "v1.0.0",
        "support_url": "https://support.example.com",
    },
})
```

### Building a `ProblemDetail` Manually

```go
detail := okapi.NewProblemDetail(400, "https://example.com/errors/bad-input", "Invalid input").
    WithInstance("/books/123").
    WithExtension("field", "name").
    WithTimestamp()
return c.AbortWithProblemDetail(detail)
```

### Supported Problem Detail Formats

| Format | Content-Type |
|--------|--------------|
| `okapi.ErrorFormatProblemJSON` | `application/problem+json` (default) |
| `okapi.ErrorFormatProblemXML` | `application/problem+xml` |

### Structured Validation Errors

Return multiple field errors at once:

```go
return c.AbortValidationErrors([]okapi.ValidationError{
    {Field: "email", Message: "must be a valid email"},
    {Field: "age",   Message: "must be ≥ 18"},
}, "validation failed")
```
