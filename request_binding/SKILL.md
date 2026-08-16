## Okapi Request Binding

Binding populates a struct from the request by inspecting struct tags and the `Content-Type`. Validation runs automatically at the end of `c.Bind` — the tags are documented in the `validation/` skill.

### Source Tags

| Tag | Source | Example |
|-----|--------|---------|
| `json:"name"` | JSON body | ``Name string `json:"name"` `` |
| `xml:"name"` | XML body | ``Name string `xml:"name"` `` |
| `yaml:"name"` | YAML body | ``Name string `yaml:"name"` `` |
| `form:"name"` | Form field / uploaded file | ``File *multipart.FileHeader `form:"file"` `` |
| `query:"page"` | Query parameter | ``Page int `query:"page"` `` |
| `path:"id"` / `param:"id"` | Path parameter | ``ID int `path:"id"` `` |
| `header:"X-Key"` | Request header | ``Key string `header:"X-API-Key"` `` |
| `cookie:"session"` | Cookie | ``Sess string `cookie:"session"` `` |

Source tags combine on one field — the body is decoded first, then path/query/form/header/cookie values are overlaid:

```go
type BookInput struct {
    ID    int    `json:"id" path:"id"`
    Name  string `json:"name" form:"name" query:"name"`
    Price int    `json:"price" form:"price" query:"price"`
}
```

### Two Binding Styles

**1. Flat binding** — body fields sit alongside query/header/cookie/path fields in one struct.

**2. Body-field binding (recommended)** — a field named `Body` (or tagged `json:"body"`) holds the payload; sibling fields carry metadata. The presence of such a field switches `c.Bind` to this mode.

```go
type CreateBookRequest struct {
    Body struct {                                        // request payload
        Name  string `json:"name" required:"true" minLength:"2"`
        Price int    `json:"price" required:"true" min:"5"`
    }

    BookID    string   `path:"bookId" required:"true"`   // path parameter
    Tags      []string `query:"tags"`                    // query parameter
    APIKey    string   `header:"X-API-Key" required:"true"` // header
    SessionID string   `cookie:"SessionID"`              // cookie
}
```

### Binding Methods on `*Context`

```go
c.Bind(&v) error              // auto: content-type body decode + tag overlay + validation
c.B(&v) error                 // shortcut for Bind
c.ShouldBind(&v) (bool, error) // same as Bind, plus an ok flag
c.BindJSON(&v) error
c.BindXML(&v) error
c.BindYAML(&v) error
c.BindProtoBuf(msg proto.Message) error
c.BindQuery(&v) error
c.BindForm(&v) error
c.BindMultipart(&v) error     // multipart/form-data, including file fields
```

`c.Bind` picks the body decoder from `Content-Type`:

| Content-Type | Decoder |
|--------------|---------|
| `application/json` | JSON |
| `application/xml` | XML |
| `application/yaml`, `text/yaml`, `application/x-yaml` | YAML |
| `application/protobuf` | Protobuf (target must implement `proto.Message`) |
| `multipart/form-data` | Multipart (handled by `BindMultipart`) |

The bind target must be a **non-nil pointer to a struct**.

### File Uploads

`BindMultipart` (and `c.Bind` on a multipart request) fills these field types from `form:` tags:

```go
type UploadRequest struct {
    Title  string                  `form:"title" required:"true"`
    File   *multipart.FileHeader   `form:"file"`
    Files  []*multipart.FileHeader `form:"files"`   // multiple files under one key
}
```

Direct access without binding:

```go
fh, err := c.FormFile("file")
c.SetMaxMultipartMemory(64 << 20)   // per-request override (app default: 32 MB)
c.MaxMultipartMemory()              // current limit
```

Set the app-wide limit with `okapi.WithMaxMultipartMemory(max int64)`.

### Typed Handlers (Bind + Validate Automatically)

```go
okapi.H(func(c *okapi.Context, in *Input) error { ... })       // shorthand
okapi.Handle(func(c *okapi.Context, in *Input) error { ... })
okapi.HandleIO(func(c *okapi.Context, in *Input) (*Output, error) { ... })
okapi.HandleO(func(c *okapi.Context) (*Output, error) { ... })
```

Pair them with `route.WithInput(...)`, `route.WithOutput(...)`, or `route.WithIO(...)` so the schemas reach OpenAPI.

### Manual Value Access

```go
c.Param("id") / c.PathParam("id")
c.Query("page")
c.QueryArray("tags")   // repeated (?tags=a&tags=b) and comma-separated (?tags=a,b)
c.QueryMap()
c.Form("name") / c.FormValue("name")
c.Header("X-API-Key") / c.Headers()
c.Cookie("session")    // (string, error)
```

### Error Handling

```go
o.Post("/books", func(c *okapi.Context) error {
    var in CreateBookRequest
    if err := c.Bind(&in); err != nil {
        return c.AbortBadRequest("Invalid request body", err)
    }
    return c.Created(in.Body)
})
```

Binding and validation failures are returned as a single `error`; convert it with an `Abort*` helper, or return structured field errors via `c.AbortValidationErrors(...)`.
