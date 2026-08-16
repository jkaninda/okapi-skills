## Okapi OpenAPI Documentation

Okapi generates OpenAPI from your routes and serves three interactive UIs. The default document is **OpenAPI 3.1** (JSON Schema 2020-12); a **3.0** view is served alongside for legacy consumers.

### Default Endpoints

| Route | Content |
|-------|---------|
| `/docs` | Selected UI (Swagger UI by default) |
| `/swagger` | Swagger UI |
| `/redoc` | ReDoc |
| `/scalar` | Scalar API Reference |
| `/openapi.json` | OpenAPI **3.1** spec (JSON) |
| `/openapi.yaml` | OpenAPI **3.1** spec (YAML) |
| `/openapi-3.0.json` | OpenAPI **3.0** spec (JSON) |
| `/openapi-3.0.yaml` | OpenAPI **3.0** spec (YAML) |

### Enabling Docs

```go
o := okapi.Default()                // docs ON by default
o := okapi.New().WithOpenAPIDocs()  // opt in for New()
o.WithOpenAPIDisabled()             // disable (routes return 404)
```

The document is rebuilt lazily, so route changes (enable/disable, new webhooks) show up on the next request.

### Choosing the UI

```go
o.WithOpenAPIDocs(okapi.OpenAPI{
    Title: "My API",
    UI:    okapi.ScalarUI, // SwaggerUI (default) | RedocUI | ScalarUI
})

// Chainable equivalent
o := okapi.New().WithOpenAPIDocs().WithDocUI(okapi.ScalarUI)
// Or as an option: okapi.New(okapi.WithDocUI(okapi.RedocUI))
```

Every UI stays reachable at its own route. `StrictDocUI: true` registers **only** `/docs`; the per-UI routes then 404:

```go
o.WithOpenAPIDocs(okapi.OpenAPI{
    Title:       "My API",
    UI:          okapi.ScalarUI,
    StrictDocUI: true,
})
```

### Full Configuration

```go
o.WithOpenAPIDocs(okapi.OpenAPI{
    Title:          "Example API",
    Version:        "1.0.0",
    Summary:        "Short summary",            // OpenAPI >= 3.1 only
    Description:    "Example API description",
    TermsOfService: "https://example.com/terms",
    Favicon:        "https://example.com/favicon.png", // favicon for the doc UIs
    Servers:        okapi.Servers{{URL: "https://api.example.com", Description: "Production"}},
    License:        okapi.License{Name: "Apache 2.0", Identifier: "Apache-2.0"}, // SPDX → 3.1 only
    Contact:        okapi.Contact{Name: "API Support", Email: "support@example.com", URL: "https://example.com"},
    ExternalDocs:   &okapi.ExternalDocs{URL: "https://docs.example.com", Description: "Docs"},
    SecuritySchemes: okapi.SecuritySchemes{
        {Name: "basicAuth", Type: "http", Scheme: "basic"},
        {Name: "bearerAuth", Type: "http", Scheme: "bearer", BearerFormat: "JWT"},
        {Name: "apiKey", Type: "apiKey", In: "header", Description: "API key header"},
        {
            Name: "OAuth2", Type: "oauth2",
            Flows: &okapi.OAuthFlows{
                AuthorizationCode: &okapi.OAuthFlow{
                    AuthorizationURL: "https://auth.example.com/authorize",
                    TokenURL:         "https://auth.example.com/token",
                    RefreshURL:       "https://auth.example.com/refresh",
                    Scopes:           map[string]string{"read": "Read access", "write": "Write access"},
                },
            },
        },
    },
})
```

`OAuthFlows` also accepts `Implicit`, `Password`, and `ClientCredentials`.

`License.Identifier` (SPDX) is mutually exclusive with `License.URL`: when set it is emitted only on the 3.1 document, where `URL` is dropped; it never appears on the 3.0 document.

### Route Documentation Options

```go
okapi.DocSummary("List books")
okapi.DocDescription("Detailed description")
okapi.DocOperationId("list-books")
okapi.DocTag("Books") / okapi.DocTags("Books", "Public")
okapi.DocPathParam(name, typ, desc)
okapi.DocPathParamWithDefault(name, typ, desc, defaultValue)
okapi.DocQueryParam(name, typ, desc, required)
okapi.DocQueryParamWithDefault(name, typ, desc, required, defaultValue)
okapi.DocHeader(name, typ, desc, required)
okapi.DocHeaderWithDefault(name, typ, desc, required, defaultValue)
okapi.DocResponseHeader(name, typ, ...desc)
okapi.DocRequestBody(&BookRequest{})
okapi.DocResponse(&Book{})              // 200
okapi.DocResponse(201, &Book{})         // explicit status
okapi.DocErrorResponse(400, &ErrorResponse{}) // deprecated → DocResponse(400, ...)
okapi.DocBearerAuth() / okapi.DocBasicAuth()
okapi.DocDeprecated() / okapi.DocHide()
okapi.UseMiddleware(mw1, mw2)           // middleware as a RouteOption
```

Short aliases without the `Doc` prefix: `Summary`, `Description`, `OperationId`, `Tag`, `Tags`, `Deprecated`, `Hide`, plus the schema options `Request(v)`, `Response(v)`, and `WithIO(req, res)`.

```go
o.Get("/books", listBooks,
    okapi.DocSummary("List books"),
    okapi.DocTags("Books"),
    okapi.DocQueryParam("page", "int", "Page number", false),
    okapi.DocResponse(&BooksResponse{}),
)
```

### Fluent DocBuilder

```go
o.Post("/books", createBook, okapi.Doc().
    Summary("Create a book").
    Description("Adds a book to the catalog").
    OperationId("createBook").
    Tags("Books").
    BearerAuth().
    PathParam("id", "int", "Book ID").
    QueryParam("expand", "string", "Expand related resources", false).
    QueryParamWithDefault("limit", "int", "Page size", false, 20).
    Header("X-Tenant", "string", "Tenant ID", true).
    ResponseHeader("X-Request-ID", "string", "Request ID").
    RequestBody(BookRequest{}).
    Response(201, Book{}).
    Response(400, okapi.ErrorResponse{}).
    Deprecated().
    Build()) // or .AsOption() — identical
```

`Doc()` also offers `Hide()`, `HeaderWithDefault(...)`, `PathParamWithDefault(...)`, and the deprecated `ErrorResponse(status, v)`.

### Schemas from Structs

Most documentation comes from struct tags — see the `validation/` skill for the full tag list (`required`, `minLength`, `enum`, `format`, `example`, `readOnly`, `hidden`, …).

```go
route := o.Post("/books", handler)
route.WithInput(&CreateBookRequest{})   // request schema (also binds + validates)
route.WithOutput(&BookResponse{})       // response schema
route.WithIO(&CreateBookRequest{}, &BookResponse{}) // both; either may be nil
route.WithSecurity(map[string][]string{"bearerAuth": {}})
route.Deprecated()
route.Hide()
```

`Request(v)` field mapping: a field named `Body` (or tagged `json:"body"`) is the request body; `path:`/`param:`, `query:`, `header:`, and `cookie:` tags become parameters.

`Response(v)` field mapping: `Status` is the status code (default 200), `Body` is the payload, `header:`/`cookie:` tagged fields become response headers/cookies.

### Reusable Component Schemas

```go
err := o.RegisterSchemas(map[string]*okapi.SchemaInfo{
    "fieldNames": {
        Schema: openapi3.NewSchemaRef("", openapi3.NewStringSchema().WithEnum([]string{"fldA", "fldB"})),
    },
})

// Reference it from a parameter
o.Get("/example", handler,
    okapi.DocQueryParamWithDefault("fields", "enum", "Fields", false,
        openapi3.NewSchemaRef("#/components/schemas/fieldNames", nil)),
)
```

Registering a name twice returns an error. Schemas can also be supplied up front via `OpenAPI.ComponentSchemas`.

### Group-Level Documentation

```go
api := o.Group("/api", jwtAuth.Middleware).
    WithTags([]string{"Books"}).                       // tags inherited by routes
    WithBearerAuth().                                   // bearerAuth requirement
    WithSecurity([]map[string][]string{{"bearerAuth": {}}}).
    Deprecated()                                        // mark every route deprecated

// Tag with a description, emitted at the spec root
api.WithTagInfo(okapi.GroupTag{
    Name:         "Books",
    Description:  "Book catalogue operations",
    ExternalDocs: &okapi.ExternalDocs{URL: "https://docs.example.com/books"},
})
```

### Per-Route Security

```go
var bearerSecurity = []map[string][]string{{"bearerAuth": {}}}
o.Get("/books", handler).WithSecurity(bearerSecurity...)

// Or on a RouteDefinition
okapi.RouteDefinition{
    Method:   http.MethodGet,
    Path:     "/books",
    Handler:  handler,
    Security: []map[string][]string{{"bearerAuth": {}}},
}
```

### OpenAPI 3.1 vs 3.0

The 3.1 document is derived from the 3.0 base and adds:

- **Type-array nullability** — pointer fields render as `nullable: true` in 3.0 and `type: ["string", "null"]` in 3.1.
- **`jsonSchemaDialect`** — JSON Schema 2020-12.
- **SPDX license identifier** — `License.Identifier`, 3.1 only.
- **`const`** — the `const:"value"` struct tag becomes a JSON Schema `const` in 3.1.
- **Webhooks** — `o.Webhook(...)`, under the `webhooks` field, 3.1 only.
- **`summary`** — `OpenAPI.Summary` on the Info object, 3.1 only.

Pin 3.0 consumers to `/openapi-3.0.json` or `/openapi-3.0.yaml`.

### Webhooks (Documentation-Only)

A webhook describes an outbound callback your API may send. It is **not** registered on the router and never receives traffic.

```go
o.Webhook("newBook", http.MethodPost,
    okapi.DocSummary("Notifies subscribers about a newly added book"),
    okapi.DocRequestBody(Book{}),
    okapi.DocResponse(200, okapi.M{"received": true}),
)
```

### Hiding Routes

```go
o.Get("/internal/metrics", handler).Hide()
o.Get("/internal/metrics", handler, okapi.DocHide())
```

Field-level: tag a struct field `hidden:"true"` to keep it out of generated schemas.

### Standard Handlers

Routes registered with `HandleStd` / `HandleHTTP` appear in the document and accept the same options, but Okapi cannot infer their schemas — declare them explicitly with `DocRequestBody` / `DocResponse`, or hide them.
