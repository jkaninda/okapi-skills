## Okapi Error Handling

Handlers return `error`. The `Abort*` helpers write a response **through the configured `ErrorHandler`** and return an error you propagate; the `Error*` helpers write a fixed JSON envelope directly, bypassing the handler. Three formats ship in the box: the default JSON envelope, a fully custom handler, and RFC 7807 Problem Details.

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

Default response body (`okapi.ErrorResponse`):

```json
{
  "code": 400,
  "message": "Invalid request body",
  "details": "field Name is required",
  "timestamp": "2026-08-16T21:34:17+01:00"
}
```

Always propagate the returned error (`return c.AbortX(...)`). Writing another body afterwards is a no-op — see "Write-Once Semantics" in the `response/` skill.

### Abort Methods

Signature: `c.AbortXxx(msg string, err ...error) error`. All route through the configured error handler.

**4xx**

| Method | Status |
|--------|--------|
| `AbortBadRequest` | 400 |
| `AbortUnauthorized` | 401 |
| `AbortPaymentRequired` | 402 |
| `AbortForbidden` | 403 |
| `AbortNotFound` | 404 |
| `AbortMethodNotAllowed` | 405 |
| `AbortNotAcceptable` | 406 |
| `AbortProxyAuthRequired` | 407 |
| `AbortRequestTimeout` | 408 |
| `AbortConflict` | 409 |
| `AbortGone` | 410 |
| `AbortLengthRequired` | 411 |
| `AbortPreconditionFailed` | 412 |
| `AbortRequestEntityTooLarge` | 413 |
| `AbortRequestURITooLong` | 414 |
| `AbortUnsupportedMediaType` | 415 |
| `AbortRequestedRangeNotSatisfiable` | 416 |
| `AbortExpectationFailed` | 417 |
| `AbortTeapot` | 418 |
| `AbortMisdirectedRequest` | 421 |
| `AbortValidationError` | 422 |
| `AbortLocked` | 423 |
| `AbortFailedDependency` | 424 |
| `AbortTooEarly` | 425 |
| `AbortUpgradeRequired` | 426 |
| `AbortPreconditionRequired` | 428 |
| `AbortTooManyRequests` | 429 |
| `AbortRequestHeaderFieldsTooLarge` | 431 |
| `AbortUnavailableForLegalReasons` | 451 |

**5xx**

| Method | Status |
|--------|--------|
| `AbortInternalServerError` / `Abort(err)` | 500 |
| `AbortNotImplemented` | 501 |
| `AbortBadGateway` | 502 |
| `AbortServiceUnavailable` | 503 |
| `AbortGatewayTimeout` | 504 |
| `AbortHTTPVersionNotSupported` | 505 |
| `AbortVariantAlsoNegotiates` | 506 |
| `AbortInsufficientStorage` | 507 |
| `AbortLoopDetected` | 508 |
| `AbortNotExtended` | 510 |
| `AbortNetworkAuthenticationRequired` | 511 |

**Generic / special**

```go
c.Abort(err) error                                 // 500 from an error
c.AbortWithError(code int, err error) error        // any status, via the error handler
c.AbortWithStatus(code int, message string) error  // any status, default ErrorResponse shape
c.AbortWithJSON(code int, jsonObj any) error       // custom JSON payload
c.AbortWithProblemDetail(p *ProblemDetail) error   // RFC 7807 payload
c.AbortNotModified() error                         // 304, no body (per spec)
c.AbortValidationErrors(errs []ValidationError, msg ...string) error
c.AbortValidationErrorsWithProblemDetail(errs []ValidationError, msg ...string) error
```

### Direct Error Writes (Bypass the Error Handler)

Signature: `c.ErrorXxx(message any) error`. These write the default JSON envelope verbatim — useful when you deliberately want a fixed shape regardless of the configured handler.

```go
c.Error(code, message)             // basic error with status + message
c.ErrorBadRequest(err)             // 400 — accepts a string, an error, or any value
c.ErrorUnauthorized("no token")    // 401
c.ErrorForbidden(msg)              // 403
c.ErrorNotFound(msg)               // 404
c.ErrorConflict(msg)               // 409
c.ErrorUnprocessableEntity(msg)    // 422
c.ErrorTooManyRequests(msg)        // 429
c.ErrorInternalServerError(msg)    // 500
c.ErrorNotModified()               // 304, no body
```

The full set mirrors the `Abort*` table above (`ErrorPaymentRequired`, `ErrorTeapot`, `ErrorLoopDetected`, `ErrorNetworkAuthenticationRequired`, …).

### Custom Error Handler

```go
o.WithErrorHandler(func(c *okapi.Context, code int, message string, err error) error {
    details := ""
    if err != nil {
        details = err.Error()
    }
    return c.JSON(code, map[string]any{
        "success": false,
        "error": map[string]any{
            "code":    code,
            "message": message,
            "details": details,
        },
    })
})

o.WithDefaultErrorHandler() // reset to okapi.DefaultErrorHandler
```

Per-request override, e.g. from middleware:

```go
func legacyErrors(c *okapi.Context) error {
    if strings.HasPrefix(c.Path(), "/v1/") {
        c.SetErrorHandler(v1ErrorHandler) // overrides the global handler for this request
    }
    return c.Next()
}
```

### RFC 7807 Problem Details

```go
o := okapi.Default()
o.WithSimpleProblemDetailErrorHandler() // application/problem+json, default fields
```

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
    Format:           okapi.ErrorFormatProblemJSON, // or ErrorFormatProblemXML / ErrorFormatDefault
    TypePrefix:       "https://api.example.com/errors/",
    IncludeInstance:  true,
    IncludeTimestamp: true,
    CustomFields: map[string]any{
        "api_version": "v1.0.0",
        "support_url": "https://support.example.com",
    },
})
```

`okapi.ProblemDetailErrorHandler(cfg)` returns the same handler as a plain `ErrorHandler` value if you want to compose it yourself.

| Format constant | Content-Type |
|-----------------|--------------|
| `okapi.ErrorFormatDefault` | `application/json` (standard `ErrorResponse`) |
| `okapi.ErrorFormatProblemJSON` | `application/problem+json` |
| `okapi.ErrorFormatProblemXML` | `application/problem+xml` |

### Building a `ProblemDetail` Manually

```go
detail := okapi.NewProblemDetail(400, "https://example.com/errors/bad-input", "Invalid input").
    WithInstance("/books/123").
    WithExtension("field", "name").
    WithTimestamp()

return c.AbortWithProblemDetail(detail)
```

`ProblemDetail` fields: `Type`, `Title`, `Status`, `Detail`, `Instance`, plus `Extensions` (flattened into the JSON object by a custom marshaller).

### Structured Validation Errors

```go
return c.AbortValidationErrors([]okapi.ValidationError{
    {Field: "email", Message: "must be a valid email", Value: input.Email},
    {Field: "age",   Message: "must be >= 18"},
}, "validation failed")
```

```json
{
  "code": 422,
  "message": "validation failed",
  "timestamp": "2026-08-16T21:34:17+01:00",
  "errors": [
    {"field": "email", "message": "must be a valid email", "value": "not-an-email"},
    {"field": "age",   "message": "must be >= 18"}
  ]
}
```

`AbortValidationErrors` always uses this `ValidationErrorResponse` shape, even when a custom error handler is configured — it has a fixed contract. Use `AbortValidationErrorsWithProblemDetail` for the RFC 7807 equivalent.

### Fallback Handlers

```go
o.NoRoute(func(c *okapi.Context) error { return c.AbortNotFound("Custom 404 - Not found") })
o.NoMethod(func(c *okapi.Context) error { return c.AbortMethodNotAllowed("Custom 405") })
```

### Errors from Standard Handlers

A `net/http` handler registered with `HandleStd` / `HandleHTTP` has no `*Context` and cannot return an error — Okapi cannot capture failures inside it. Write the error response yourself, or convert the route to a native handler.

### Status Class Helpers

```go
okapi.IsError(code) bool        // 4xx or 5xx
okapi.IsClientError(code) bool  // 4xx
okapi.IsServerError(code) bool  // 5xx
```
