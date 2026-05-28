## Okapi Authentication & CORS

### JWT Authentication

`JWTAuth` validates the request token, optionally enforces claims, and stores claims in the context.

```go
jwtAuth := okapi.JWTAuth{
    SigningSecret: []byte("supersecret"),    // HS256 / HS384 / HS512
    RsaKey:        publicKey,                 // RS256 / RS384 / RS512
    JwksUrl:       "https://issuer.example.com/.well-known/jwks.json",
    JwksFile:      jwks,                      // okapi.LoadJWKSFromFile(...)
    Algo:          "RS256",                   // pin expected algorithm
    Audience:      "my-api",
    Issuer:        "https://issuer.example.com",
    TokenLookup:   "header:Authorization",    // also: "query:token", "cookie:jwt"
    ContextKey:    "user",                    // where to store claims
    ForwardClaims: map[string]string{
        "email": "user.email",
        "role":  "realm_access.roles.0",
    },
    ClaimsExpression: "Equals(`email_verified`, `true`) && OneOf(`user.role`, `admin`, `owner`)",
    ValidateClaims:   func(c *okapi.Context, claims jwt.Claims) error { return nil },
    OnUnauthorized:   func(c *okapi.Context) error { return c.ErrorUnauthorized("Nope") },
}
api := o.Group("/api", jwtAuth.Middleware).WithBearerAuth()
```

Configure **at least one** verification mechanism: `SigningSecret`, `RsaKey`, `JwksUrl`, or `JwksFile`.

### Claims Expression DSL

`ClaimsExpression` is a string parsed into an AST. Supported function calls:

| Function | Description |
|----------|-------------|
| `Equals(field, value)` | Exact equality (compares against scalars and arrays) |
| `Prefix(field, prefix)` | String prefix match |
| `Contains(field, val1, val2, ...)` | Field (array) contains **all** of the values |
| `OneOf(field, val1, val2, ...)` | Field equals **any** of the values |

Logical operators: `!` (NOT), `&&` (AND), `||` (OR). `&&` binds tighter than `||`.

Field names and string literals use **backticks**, and dot notation drills into nested claims:

```go
ClaimsExpression: "Equals(`email_verified`, `true`) && (OneOf(`user.role`, `admin`, `owner`) || Contains(`tags`, `vip`))"
```

Programmatic equivalents (also exported, useful for composing expressions in code):

```go
expr := okapi.And(
    okapi.Equals("email_verified", "true"),
    okapi.Or(
        okapi.OneOf("user.role", "admin", "owner"),
        okapi.Contains("tags", "vip"),
    ),
)
```

### Forwarding Claims to the Context

```go
jwtAuth.ForwardClaims = map[string]string{
    "email": "user.email",
    "role":  "user.role",
    "name":  "user.name",
}

// In a handler:
email := c.GetString("email")
role  := c.GetString("role")
```

### Custom Claim Validation

```go
jwtAuth.ValidateClaims = func(c *okapi.Context, claims jwt.Claims) error {
    mc, ok := claims.(jwt.MapClaims)
    if !ok {
        return errors.New("invalid claims type")
    }
    if v, _ := mc["email_verified"].(bool); !v {
        return errors.New("email not verified")
    }
    return nil
}
```

### Custom Unauthorized Handler

```go
jwtAuth.OnUnauthorized = func(c *okapi.Context) error {
    return c.ErrorUnauthorized("Custom unauthorized payload")
}
```

### Token Generation

```go
token, err := okapi.GenerateJwtToken(secret, jwt.MapClaims{"sub": "123"}, 24*time.Hour)
```

### JWKS Loading

```go
jwks, err := okapi.LoadJWKSFromFile("path/to/jwks.json") // also accepts base64 string
jwtAuth.JwksFile = jwks
```

### Basic Authentication

```go
basicAuth := okapi.BasicAuth{
    Username:   "admin",
    Password:   "secret",
    Realm:      "Admin Area",
    ContextKey: "user", // default "username"
}

// Global, group, or per-route
o.Use(basicAuth.Middleware)
admin := o.Group("/admin", basicAuth.Middleware).WithBasicAuth()
o.Get("/dashboard", h).Use(basicAuth.Middleware)
```

### CORS

```go
o.WithCORS(okapi.Cors{
    AllowedOrigins:   []string{"https://app.example.com", "https://*.example.com", "*"},
    AllowedHeaders:   []string{"Content-Type", "Authorization"},
    AllowMethods:     []string{"GET", "POST", "PUT", "DELETE"},
    ExposeHeaders:    []string{"X-Request-ID"},
    Headers:          map[string]string{"X-Frame-Options": "DENY"}, // extra response headers
    MaxAge:           3600,
    AllowCredentials: true,
})
```

Wildcards:
- `"*"` — allow any origin (cannot be combined with `AllowCredentials: true` if you need credentialed requests — the origin is echoed verbatim instead).
- `"https://*.example.com"` — scheme + wildcard subdomain.
- Exact origins are matched case-insensitively.

Real preflights (OPTIONS + `Access-Control-Request-Method`) get an automatic 204 from the CORS handler; plain OPTIONS requests fall through to your handler.
