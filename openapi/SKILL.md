## Okapi OpenAPI Documentation

Okapi auto-generates OpenAPI from your routes and serves it through three interactive UIs out of the box. The default document is **OpenAPI 3.1** (JSON Schema 2020-12); a **3.0** view is also served for legacy consumers.

### Default Endpoints

| Route | Content |
|-------|---------|
| `/docs` | Selected UI (default: Swagger UI) |
| `/swagger` | Swagger UI |
| `/redoc` | ReDoc |
| `/scalar` | Scalar API Reference |
| `/openapi.json` | OpenAPI **3.1** spec (JSON) |
| `/openapi.yaml` | OpenAPI **3.1** spec (YAML) |
| `/openapi-3.0.json` | OpenAPI **3.0** spec (JSON) |
| `/openapi-3.0.yaml` | OpenAPI **3.0** spec (YAML) |

### Enabling Docs

```go
o := okapi.Default()                  // Docs ON by default at /docs
o := okapi.New().WithOpenAPIDocs()     // Opt-in for New()
o.WithOpenAPIDisabled()                // Disable at runtime
```

### Choosing the Documentation UI

```go
// Field on OpenAPI config
o.WithOpenAPIDocs(okapi.OpenAPI{
    Title: "My API",
    UI:    okapi.ScalarUI, // SwaggerUI (default) | RedocUI | ScalarUI
})

// Chainable
o := okapi.New().WithOpenAPIDocs().WithDocUI(okapi.ScalarUI)
```

By default every UI stays reachable at its own route. Set `StrictDocUI: true` to register **only** the selected UI — the other UI routes then return 404:

```go
o.WithOpenAPIDocs(okapi.OpenAPI{
    Title:       "My API",
    UI:          okapi.ScalarUI,
    StrictDocUI: true, // only /docs and /scalar are served
})
```

### Full OpenAPI Configuration

```go
o.WithOpenAPIDocs(okapi.OpenAPI{
    Title:        "Example API",
    Version:      "1.0.0",
    Description:  "Example API description",
    Servers:      okapi.Servers{{URL: "https://api.example.com"}},
    License:      okapi.License{Name: "Apache 2.0", Identifier: "Apache-2.0"}, // SPDX → 3.1 only
    Contact:      okapi.Contact{Name: "API Support", Email: "support@example.com"},
    ExternalDocs: &okapi.ExternalDocs{URL: "https://docs.example.com", Description: "Docs"},
    SecuritySchemes: okapi.SecuritySchemes{
        {Name: "basicAuth", Type: "http", Scheme: "basic"},
        {Name: "bearerAuth", Type: "http", Scheme: "bearer", BearerFormat: "JWT"},
        {
            Name: "OAuth2", Type: "oauth2",
            Flows: &okapi.OAuthFlows{
                AuthorizationCode: &okapi.OAuthFlow{
                    AuthorizationURL: "https://auth.example.com/authorize",
                    TokenURL:         "https://auth.example.com/token",
                    Scopes:           map[string]string{"read": "Read access", "write": "Write"},
                },
            },
        },
    },
})
```

### Route Documentation Options (Composable)

```go
okapi.DocSummary("List books")
okapi.DocDescription("Detailed description")
okapi.DocOperationId("list-books")
okapi.DocTags("Books", "Public")
okapi.DocPathParam(name, type, description)
okapi.DocPathParamWithDefault(name, type, description, defaultValue)
okapi.DocQueryParam(name, type, description, required)
okapi.DocQueryParamWithDefault(name, type, description, required, defaultValue)
okapi.DocHeader(name, type, description, required)
okapi.DocHeaderWithDefault(name, type, description, required, defaultValue)
okapi.DocRequestBody(&BookRequest{})
okapi.DocResponse(&Book{})                  // 200
okapi.DocResponse(201, &Book{})             // explicit status
okapi.DocErrorResponse(400, &ErrorResponse{})
okapi.DocResponseHeader(name, type, ...description)
okapi.DocBearerAuth()
okapi.DocBasicAuth()
okapi.DocDeprecated()
okapi.DocHide()
okapi.Request(&BookRequest{})
okapi.Response(&BookResponse{})
okapi.WithIO(&Req{}, &Resp{})
```

### Fluent DocBuilder

```go
okapi.Doc().
    Summary("Create a book").
    Description("Adds a book to the catalog").
    OperationId("createBook").
    Tags("Books").
    BearerAuth().
    PathParam("id", "int", "Book ID").
    QueryParam("expand", "string", "Expand related resources", false).
    Header("X-Tenant", "string", "Tenant ID", true).
    ResponseHeader("X-Request-ID", "string", "Request ID").
    RequestBody(BookRequest{}).
    Response(201, Book{}).
    Response(400, ErrorResponse{}).
    Deprecated().
    Build()
```

### Per-Route Schema Helpers

```go
route.WithIO(&BookRequest{}, &BookResponse{})  // both
route.WithInput(&BookRequest{})                 // request only
route.WithOutput(&BookResponse{})               // response only
route.WithSecurity(map[string][]string{"bearerAuth": {}})
route.Deprecated()
route.Hide()                                    // hide from docs
```

### Group-Level

```go
api := o.Group("/api", jwtMiddleware).
    WithTags([]string{"Books"}).        // OpenAPI tags
    WithBearerAuth().                    // adds bearerAuth scheme requirement
    WithBasicAuth().                     // adds basicAuth scheme requirement
    WithSecurity(bearerAuthSecurity).    // explicit security requirements
    Deprecated()                          // mark all routes deprecated
```

### Applying Security to a Single Route

```go
var bearerSecurity = []map[string][]string{{"bearerAuth": {}}}
o.Get("/books", handler).WithSecurity(bearerSecurity...)
```

### OpenAPI 3.1 Features (vs 3.0)

The 3.1 document is derived from the 3.0 base and adds:

- **Type-array nullability** — pointer fields render as `nullable: true` in 3.0 and as `type: ["string", "null"]` in 3.1.
- **`jsonSchemaDialect`** — set to JSON Schema 2020-12.
- **SPDX license identifier** — set `License.Identifier` (e.g. `"Apache-2.0"`); appears only on the 3.1 document and is mutually exclusive with `License.URL`.
- **`const` schema** — the `const:"value"` struct tag becomes a JSON Schema `const` in 3.1.
- **Webhooks** — declared with `o.Webhook(...)`; appear only under the `webhooks` field of the 3.1 document.

### Webhooks (Documentation-Only)

A webhook describes an outbound callback your API may send. It is **not** added to the router; it only appears in the OpenAPI 3.1 document.

```go
o.Webhook("newBook", http.MethodPost,
    okapi.DocSummary("Notifies subscribers about a newly added book"),
    okapi.DocRequestBody(Book{}),
    okapi.DocResponse(200, okapi.M{"received": true}),
)
```

### Hiding a Route from Docs

```go
o.Get("/internal/metrics", handler).Hide()
// or
o.Get("/internal/metrics", handler, okapi.DocHide())
```
