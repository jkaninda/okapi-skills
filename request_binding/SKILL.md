## Okapi Request Binding & Validation

### Struct Tags

| Tag | Source | Example |
|-----|--------|---------|
| `json:"name"` | JSON body | `Name string \`json:"name"\`` |
| `xml:"name"` | XML body | `Name string \`xml:"name"\`` |
| `query:"name"` | Query parameter | `Page int \`query:"page"\`` |
| `path:"id"` / `param:"id"` | Path parameter | `ID int \`path:"id"\`` |
| `header:"X-Key"` | HTTP header | `Key string \`header:"X-Key"\`` |
| `cookie:"session"` | Cookie | `Sess string \`cookie:"session"\`` |
| `form:"file"` | Form field / file | `File string \`form:"file"\`` |

### Binding Methods on Context

```go
c.Bind(&v)            // Auto-bind (Body field or flat struct)
c.B(&v)               // Shortcut for Bind
c.ShouldBind(&v)      // Returns (bool, error)
c.BindJSON(&v)        // JSON body
c.BindXML(&v)         // XML body
c.BindYAML(&v)        // YAML body
c.BindProtoBuf(&msg)  // Protobuf body
c.BindQuery(&v)       // Query params
c.BindForm(&v)        // Form data
c.BindMultipart(&v)   // Multipart form
```

### Body Field Pattern

Separate payload from metadata:

```go
type CreateBookRequest struct {
    Body   Book   `json:"body"`                          // Request payload
    ID     int    `param:"id" query:"id"`                // Path or query param
    APIKey string `header:"X-API-Key" required:"true"`   // Header
}
```

### Validation Tags

| Tag | Description | Example |
|-----|-------------|---------|
| `required:"true"` | Field is required | `Name string \`required:"true"\`` |
| `min:"5"` | Minimum numeric value | `Price int \`min:"5"\`` |
| `max:"100"` | Maximum numeric value | `Price int \`max:"100"\`` |
| `minLength:"3"` | Minimum string length | `Name string \`minLength:"3"\`` |
| `maxLength:"50"` | Maximum string length | `Name string \`maxLength:"50"\`` |
| `pattern:"^[A-Z]"` | Regex pattern | `Code string \`pattern:"^[A-Z]+$"\`` |
| `enum:"a,b,c"` | Allowed values | `Status string \`enum:"active,inactive"\`` |
| `default:"val"` | Default value | `Status string \`default:"active"\`` |
| `format:"email"` | Format validation | `Email string \`format:"email"\`` |
| `multipleOf:"5"` | Must be multiple of | `Qty int \`multipleOf:"5"\`` |
| `minItems:"1"` | Minimum slice length | `Tags []string \`minItems:"1"\`` |
| `maxItems:"10"` | Maximum slice length | `Tags []string \`maxItems:"10"\`` |
| `uniqueItems:"true"` | Unique slice items | `Tags []string \`uniqueItems:"true"\`` |
| `deprecated:"true"` | Mark as deprecated | `Old string \`deprecated:"true"\`` |
| `description:"text"` | Field description | `Name string \`description:"User name"\`` |
| `example:"value"` | Example value | `Name string \`example:"John"\`` |

### Supported Formats

`email`, `date-time` (RFC3339), `date` (YYYY-MM-DD), `duration`, `ipv4`, `ipv6`, `hostname`, `uri`, `uuid`, `regex`
