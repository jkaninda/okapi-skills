## Okapi OpenAPI Documentation

### Endpoints (when enabled)

- `/docs` - Swagger UI
- `/redoc` - ReDoc
- `/openapi.json` - OpenAPI spec

### Route Documentation Options (Composable)

```go
okapi.DocSummary("List books")
okapi.DocDescription("Detailed description")
okapi.DocOperationId("list-books")
okapi.DocTags("Books", "Public")
okapi.DocPathParam(name, type, description)
okapi.DocQueryParam(name, type, description, required)
okapi.DocHeader(name, type, description, required)
okapi.DocRequestBody(&BookRequest{})
okapi.DocResponse(&Book{})
okapi.DocResponse(201, &Book{})
okapi.DocErrorResponse(400, &ErrorResponse{})
okapi.DocResponseHeader(name, type, ...description)
okapi.DocBearerAuth()
okapi.DocBasicAuth()
okapi.DocDeprecated()
okapi.DocHide()
okapi.Request(&BookRequest{})
okapi.Response(&BookResponse{})
```

### Fluent DocBuilder

```go
okapi.Doc().
    Summary("Create a book").
    Tags("Books").
    BearerAuth().
    RequestBody(BookRequest{}).
    Response(201, Book{}).
    Response(400, ErrorResponse{}).
    ResponseHeader("X-Request-ID", "string", "Request ID").
    Build()
```

### Fluent Route Methods

```go
route.WithIO(&BookRequest{}, &BookResponse{})
route.WithInput(&BookRequest{})
route.WithOutput(&BookResponse{})
```

### OpenAPI Configuration

```go
app.WithOpenAPIDocs(okapi.OpenAPI{
    Title:           "My API",
    Version:         "1.0.0",
    Servers:         okapi.Servers{{URL: "https://api.example.com"}},
    License:         okapi.License{Name: "MIT"},
    Contact:         okapi.Contact{Name: "Team", Email: "team@example.com"},
    SecuritySchemes: schemes,
    ExternalDocs:    &okapi.ExternalDocs{URL: "https://docs.example.com"},
})
```
