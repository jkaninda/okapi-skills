## Okapi Request Binding & Validation

### Source Tags

| Tag | Source | Example |
|-----|--------|---------|
| `json:"name"` | JSON body | `Name string \`json:"name"\`` |
| `xml:"name"` | XML body | `Name string \`xml:"name"\`` |
| `yaml:"name"` | YAML body | `Name string \`yaml:"name"\`` |
| `query:"name"` | Query parameter | `Page int \`query:"page"\`` |
| `path:"id"` / `param:"id"` | Path parameter | `ID int \`path:"id"\`` |
| `header:"X-Key"` | HTTP header | `Key string \`header:"X-Key"\`` |
| `cookie:"session"` | Cookie | `Sess string \`cookie:"session"\`` |
| `form:"file"` | Form field / file | `File string \`form:"file"\`` |

You can combine source tags on the same field — Okapi tries each source until a value is found:

```go
type BookInput struct {
    ID    int    `json:"id" path:"id"`
    Name  string `json:"name" form:"name" query:"name"`
    Price int    `json:"price" form:"price" query:"price"`
}
```

### Binding Methods on Context

```go
c.Bind(&v)            // Auto-bind (Body field or flat struct) + validate
c.B(&v)               // Shortcut for Bind
c.ShouldBind(&v)      // Returns (bool, error) - no abort on failure
c.BindJSON(&v)        // JSON body
c.BindXML(&v)         // XML body
c.BindYAML(&v)        // YAML body
c.BindProtoBuf(&msg)  // Protobuf body
c.BindQuery(&v)       // Query params
c.BindForm(&v)        // Form data
c.BindMultipart(&v)   // Multipart form
```

### Body Field Pattern

Separate the payload from metadata using a `Body` field. Path/query/header tags sit outside the Body, JSON tags sit inside it.

```go
type CreateBookRequest struct {
    Body Book `json:"body"`                    // Request payload (JSON)

    ID     int    `param:"id" query:"id"`       // Path or query param
    APIKey string `header:"X-API-Key" required:"true"`
}
```

### Typed Handlers (Auto-Bind + Validate)

```go
// Auto-bind input
okapi.H(func(c *okapi.Context, in *Input) error { ... })
okapi.Handle(func(c *okapi.Context, in *Input) error { ... })

// Input + typed output
okapi.HandleIO(func(c *okapi.Context, in *Input) (*Output, error) { ... })

// Output only
okapi.HandleO(func(c *okapi.Context) (*Output, error) { ... })
```

### Validation Tags

| Tag | Applies to | Description |
|-----|------------|-------------|
| `required:"true"` | any | Field must be present and non-zero |
| `default:"v"` | any | Assigned when missing/empty |
| `description:"..."` | any | OpenAPI description |
| `example:"v"` | any | OpenAPI example |
| `deprecated:"true"` | any | Mark deprecated in docs |
| `min:"5"` | number | ≥ 5 |
| `max:"100"` | number | ≤ 100 |
| `exclusiveMin:"0"` | number | > 0 |
| `exclusiveMax:"100"` | number | < 100 |
| `multipleOf:"5"` | number | divisible by 5 |
| `minLength:"3"` | string | length ≥ 3 |
| `maxLength:"50"` | string | length ≤ 50 |
| `pattern:"^[A-Z]+$"` | string / []string | regex |
| `format:"email"` | string / []string | format validation (see below) |
| `enum:"a,b,c"` | string / []string | one of these values |
| `const:"value"` | string / []string | must equal this value (becomes JSON Schema `const` in OAS 3.1) |
| `minItems:"1"` | slice | length ≥ 1 |
| `maxItems:"10"` | slice | length ≤ 10 |
| `uniqueItems:"true"` | slice | no duplicates |
| `minProperties:"1"` | map | size ≥ 1 |
| `maxProperties:"10"` | map | size ≤ 10 |

> **Slices:** `enum`, `const`, `format`, and `pattern` apply to **each element** of a `[]string` field. Failures are reported per index (e.g. `element [2]: ...`).
>
> **Empty values:** `enum`, `const`, `format`, and `pattern` skip empty strings — combine with `required:"true"` to also enforce presence.

### Supported Formats

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
| `email` | Valid email |
| `hostname` | Valid hostname |
| `ipv4` / `ipv6` | IP addresses |
| `mac` | MAC address |
| `cidr` | CIDR notation (`192.168.1.0/24`) |
| `uri` / `uri-reference` | Any URI (or relative reference) |
| `url` | Absolute http/https URL |
| `uuid` / `ulid` | UUID / ULID |
| `e164` / `phone` | E.164 phone (`+14155552671`) |
| `credit-card` | Luhn-valid card number |
| `semver` | Semantic version |
| `json-pointer` | RFC 6901 JSON Pointer |
| `byte` / `base64` | Base64-encoded data |

#### String content

| Format | Description |
|--------|-------------|
| `alpha` | Letters only |
| `alphanumeric` | Letters + digits |
| `numeric` | Numeric string |
| `ascii` | ASCII only |
| `lowercase` / `uppercase` | Case constraint |
| `slug` | URL slug (`my-post-123`) |
| `hexcolor` | `#RGB` or `#RRGGBB` |
| `regex` | Use with `pattern` to validate via custom regex |

### Example

```go
type CreateUserRequest struct {
    Email    string            `json:"email" required:"true" format:"email"`
    Password string            `json:"password" minLength:"8"`
    Age      int               `json:"age" exclusiveMin:"0" max:"120" default:"18"`
    Website  string            `json:"website" format:"url"`
    Kind     string            `json:"kind" const:"user"`
    Roles    []string          `json:"roles" minItems:"1" uniqueItems:"true" enum:"admin,editor,viewer"`
    Metadata map[string]string `json:"metadata" minProperties:"1" maxProperties:"10"`
    Phone    string            `json:"phone" format:"e164"`
    UUID     string            `json:"uuid" format:"uuid"`
}
```
