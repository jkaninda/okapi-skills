## Okapi Validation

Validation is declarative: struct tags drive both runtime enforcement and OpenAPI schema generation. Validation runs automatically inside `c.Bind` and inside the typed handler wrappers (`okapi.Handle`, `okapi.H`, `okapi.HandleIO`, `okapi.HandleO`).

For where values come from (JSON body, query, path, header, cookie, form), see the `request_binding/` skill.

### Constraint Tags

| Tag | Applies to | Description |
|-----|------------|-------------|
| `required:"true"` | any | Field must be present and non-zero |
| `default:"v"` | any | Assigned when the field is missing/empty |
| `min:"5"` | number | ≥ 5 |
| `max:"100"` | number | ≤ 100 |
| `exclusiveMin:"0"` | number | > 0 |
| `exclusiveMax:"100"` | number | < 100 |
| `multipleOf:"5"` | number | Divisible by 5 |
| `minLength:"3"` | string | Length ≥ 3 |
| `maxLength:"50"` | string | Length ≤ 50 |
| `pattern:"^[A-Z]+$"` | string / `[]string` | Regular expression |
| `format:"email"` | string / `[]string` | Format validation (table below) |
| `enum:"a,b,c"` | string / `[]string` | One of the listed values |
| `const:"active"` | string / `[]string` | Must equal a fixed value (JSON Schema `const` in OAS 3.1) |
| `contains:"@"` | string / `[]string` | Must contain the substring |
| `notContains:" "` | string / `[]string` | Must not contain the substring |
| `minItems:"2"` | slice | At least 2 items |
| `maxItems:"5"` | slice | At most 5 items |
| `uniqueItems:"true"` | slice | No duplicates |
| `minProperties:"1"` | map | At least 1 entry |
| `maxProperties:"10"` | map | At most 10 entries |

> **Slices:** `enum`, `const`, `format`, `pattern`, `contains`, and `notContains` apply to **each element** of a `[]string`. Failures are reported per index — `element [2]: ...`.
>
> **Empty values:** those same tags skip empty strings. Combine with `required:"true"` to also enforce presence.

### Conditional Required Tags

Requiredness that depends on **sibling fields** in the same struct. Referenced fields use their **Go field name**, not the JSON name.

| Tag | Description |
|-----|-------------|
| `requiredIf:"Type card"` | Required when sibling `Type` equals `card` |
| `requiredWith:"Pass"` | Required when any listed sibling (comma-separated) is non-empty |
| `requiredWithout:"Email"` | Required when any listed sibling (comma-separated) is empty |

```go
type Payment struct {
    Type    string `json:"type"`
    Card    string `json:"card"    requiredIf:"Type card"`
    Pass    string `json:"pass"`
    Confirm string `json:"confirm" requiredWith:"Pass"`
    Email   string `json:"email"`
    Phone   string `json:"phone"   requiredWithout:"Email"`
}
```

### Documentation-Only Tags

These shape the OpenAPI schema; they are not enforced at runtime.

| Tag | Description |
|-----|-------------|
| `description:"..."` | Property description |
| `doc:"..."` | Alternative description tag |
| `example:"v"` | Example value |
| `deprecated:"true"` | Mark the property deprecated |
| `hidden:"true"` | Omit the field from the generated schema |
| `readOnly:"true"` | Returned by the server, not sent by clients |
| `writeOnly:"true"` | Sent by clients, not returned |
| `nullable:"true"` | Nullable property (pointer fields are nullable automatically) |

### Formats

All formats apply to `string` fields and to each element of a `[]string`.

#### Date & time

| Format | Description |
|--------|-------------|
| `date` | `YYYY-MM-DD` |
| `date-time` | RFC3339 timestamp |
| `time` | RFC3339 full-time (`15:04:05Z`) |
| `duration` | Go duration (`1h30m`, `300ms`) |

#### Network, web & identifiers

| Format | Description |
|--------|-------------|
| `email` | Valid email address |
| `hostname` | Valid hostname |
| `ipv4` / `ipv6` | IP addresses |
| `mac` | MAC address |
| `cidr` | CIDR notation (`192.168.1.0/24`) |
| `uri` / `uri-reference` | Any URI / relative reference allowed |
| `url` | Absolute `http`/`https` URL |
| `uuid` / `ulid` | UUID / ULID |
| `e164` / `phone` | E.164 phone number (`+14155552671`) |
| `credit-card` | Luhn-valid card number |
| `semver` | Semantic version (`1.2.3-alpha.1`) |
| `json-pointer` | RFC 6901 JSON Pointer |
| `byte` / `base64` | Base64-encoded value |
| `base64url` | URL-safe Base64 (padded or unpadded) |
| `jwt` | JSON Web Token (three base64url segments) |
| `port` | TCP/UDP port (`1`–`65535`) |

#### String content

| Format | Description |
|--------|-------------|
| `alpha` | Letters only |
| `alphanumeric` | Letters and digits |
| `numeric` | Numeric string (`123`, `-12.5`) |
| `ascii` | ASCII only |
| `lowercase` / `uppercase` | Case constraint |
| `slug` | URL slug (`my-post-123`) |
| `hexcolor` | `#RGB` or `#RRGGBB` |
| `json` | Syntactically valid JSON string |

#### Geo & time zones

| Format | Description |
|--------|-------------|
| `latitude` | Decimal latitude, `-90`…`90` |
| `longitude` | Decimal longitude, `-180`…`180` |
| `timezone` | IANA time zone (`America/New_York`) |

#### Custom pattern

```go
Phone string `json:"phone" format:"regex" pattern:"^\\+?[1-9]\\d{1,14}$"`
```

### Example

```go
type CreateUserRequest struct {
    Email    string            `json:"email" required:"true" format:"email" example:"user@example.com"`
    Password string            `json:"password" minLength:"8" writeOnly:"true" description:"User password"`
    Age      int               `json:"age" exclusiveMin:"0" max:"120" default:"18"`
    Website  string            `json:"website" format:"url"`
    Kind     string            `json:"kind" const:"user"`
    Roles    []string          `json:"roles" minItems:"1" uniqueItems:"true" enum:"admin,editor,viewer"`
    Metadata map[string]string `json:"metadata" minProperties:"1" maxProperties:"10"`
    ID       string            `json:"id" format:"uuid" readOnly:"true"`
    Internal string            `json:"-" hidden:"true"`
}
```

### Five Ways to Validate

**1. `c.Bind()` — bind and validate inside the handler**

```go
o.Post("/users", func(c *okapi.Context) error {
    var req CreateUserRequest
    if err := c.Bind(&req); err != nil {
        return c.ErrorBadRequest(err)
    }
    return c.JSON(http.StatusOK, req)
})
```

**2. `okapi.Handle()` — typed input, manual response**

```go
o.Post("/books", okapi.Handle(func(c *okapi.Context, book *Book) error {
    book.ID = nextID()
    return c.Created(book)
}),
    okapi.DocRequestBody(&Book{}),
    okapi.DocResponse(&Book{}),
)
```

**3. `okapi.H()` — shorthand for `Handle`**

```go
o.Get("/books/{id:int}", okapi.H(func(c *okapi.Context, in *BookDetailInput) error {
    b := find(in.ID)
    if b == nil {
        return c.AbortNotFound("Book not found")
    }
    return c.OK(b)
})).WithInput(&BookDetailInput{})
```

**4. `okapi.HandleIO()` — typed input and typed output**

```go
type BookEditInput struct {
    ID   int  `json:"id" path:"id" required:"true"`
    Body Book `json:"body"`
}
type BookOutput struct {
    Status int
    Body   Book
}

o.Put("/books/{id:int}", okapi.HandleIO(func(c *okapi.Context, in *BookEditInput) (*BookOutput, error) {
    b := update(in.ID, in.Body)
    if b == nil {
        return nil, c.AbortNotFound("Book not found")
    }
    return &BookOutput{Body: *b}, nil
})).WithIO(&BookEditInput{}, &BookOutput{})
```

**5. `okapi.HandleO()` — typed output only**

```go
type BooksResponse struct {
    Body []Book `json:"books"`
}

o.Get("/books", okapi.HandleO(func(c *okapi.Context) (*BooksResponse, error) {
    return &BooksResponse{Body: books}, nil
})).WithOutput(&BooksResponse{})
```

The output struct follows the body-style convention (`Status`, `Body`, header/cookie-tagged fields). The response format comes from the client's `Accept` header, defaulting to JSON.

### Schema Helpers

| Helper | Purpose |
|--------|---------|
| `okapi.DocRequestBody(&T{})` | Document the request body schema |
| `okapi.DocResponse(&T{})` / `okapi.DocResponse(201, &T{})` | Document a response schema |
| `route.WithInput(&T{})` | Input schema (pairs with `okapi.H` / `Handle`) |
| `route.WithOutput(&T{})` | Output schema (pairs with `HandleO`) |
| `route.WithIO(&In{}, &Out{})` | Both (pairs with `HandleIO`) |

### Returning Validation Errors

```go
return c.AbortValidationErrors([]okapi.ValidationError{
    {Field: "email", Message: "must be a valid email", Value: req.Email},
    {Field: "age",   Message: "must be >= 18"},
}, "validation failed")
```

`AbortValidationErrors` always uses the `ValidationErrorResponse` shape (an `ErrorResponse` plus an `errors` array), regardless of the configured error handler. Use `AbortValidationErrorsWithProblemDetail` for the RFC 7807 form.
