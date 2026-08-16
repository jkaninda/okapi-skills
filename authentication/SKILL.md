## Okapi Authentication & CORS

### JWT Authentication

`JWTAuth` extracts the token, verifies the signature, optionally enforces claims, and stores claims in the context.

```go
jwtAuth := okapi.JWTAuth{
    SigningSecret: []byte("supersecret"),   // HMAC (HS256/384/512)
    RsaKey:        publicKey,               // RSA (RS256/384/512)
    JwksUrl:       "https://issuer.example.com/.well-known/jwks.json",
    JwksFile:      jwks,                    // from okapi.LoadJWKSFromFile(...)

    Algorithms:  []string{"RS256", "ES256"}, // accepted signing algorithms
    Audience:    "my-api",                   // expected "aud"
    Issuer:      "https://issuer.example.com", // expected "iss"
    TokenLookup: "header:Authorization,cookie:jwt", // sources, tried in order

    ContextKey: "user",                      // store the full claims under this key
    ForwardClaims: map[string]string{        // copy selected claims into the store
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

| Field | Notes |
|-------|-------|
| `SigningSecret []byte` | HMAC key. Supersedes the deprecated `SecretKey`. |
| `RsaKey *rsa.PublicKey` | RSA public key. |
| `JwksUrl string` | Remote JWKS endpoint for key discovery. |
| `JwksFile *Jwks` | Static JWKS from file or base64 (`okapi.LoadJWKSFromFile`). |
| `Algorithms []string` | Accepted algorithms. Defaults to `RS256`, `HS256`, `ES256`. Supersedes the deprecated single-valued `Algo`. |
| `Audience` / `Issuer` | Validated `aud` / `iss` claims. |
| `TokenLookup string` | Comma-separated `source:name` list; the first non-empty hit wins. Default `header:Authorization`. |
| `ContextKey string` | Key holding the full `jwt.MapClaims`. |
| `ForwardClaims map[string]string` | `contextKey -> claim.path` (dot notation, numeric indices allowed). |
| `ClaimsExpression string` | Expression DSL (below). |
| `ValidateClaims func(*Context, jwt.Claims) error` | Custom validation; supersedes the deprecated `ValidateRole`. |
| `OnUnauthorized HandlerFunc` | Custom response for any auth failure. |

**Token sources** — `header:Authorization` (a `Bearer ` prefix is stripped), `query:token`, `cookie:jwt`. Combine them:

```go
TokenLookup: "header:Authorization,query:token,cookie:jwt"
```

**Status codes** — a missing/expired/malformed token gives **401**; a token that verifies but fails `ClaimsExpression` or `ValidateClaims` gives **403**. `OnUnauthorized` overrides both.

Validate a token by hand (outside the middleware):

```go
claims, err := jwtAuth.ValidateToken(c) // (jwt.MapClaims, error)
```

### Claims Expression DSL

`ClaimsExpression` is a string parsed into an AST.

| Function | Description |
|----------|-------------|
| `Equals(field, value)` | Exact equality (compares against scalars and array members) |
| `Prefix(field, prefix)` | String prefix match |
| `Contains(field, v1, v2, ...)` | Field contains **all** listed values (substring or array membership) |
| `OneOf(field, v1, v2, ...)` | Field equals **any** listed value |

Operators: `!` (NOT), `&&` (AND), `||` (OR) — `&&` binds tighter than `||`. Field names and literals use **backticks**; dot notation drills into nested claims.

```go
ClaimsExpression: "Equals(`email_verified`, `true`) && (OneOf(`user.role`, `admin`, `owner`) || Contains(`tags`, `vip`))"
```

Programmatic equivalents (useful for composing expressions in code):

```go
expr := okapi.And(
    okapi.Equals("email_verified", "true"),
    okapi.Or(
        okapi.OneOf("user.role", "admin", "owner"),
        okapi.Contains("tags", "vip"),
    ),
)
ok, err := expr.Evaluate(claims)          // claims is jwt.MapClaims

parsed, err := okapi.ParseExpression("Prefix(`sub`, `user_`) && !Equals(`banned`, `true`)")
```

Types: `AndExpr`, `OrExpr`, `NotExpr`, `EqualsExpr`, `PrefixExpr`, `ContainsExpr`, `OneOfExpr` — all implement `Expression`.

### Reading Claims in a Handler

```go
// Full claims via ContextKey
if v, ok := c.Get("user"); ok {
    claims := v.(jwt.MapClaims)
    sub, _ := claims["sub"].(string)
}

// Individual forwarded claims (stored as strings)
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

### Custom Unauthorized Response

```go
jwtAuth.OnUnauthorized = func(c *okapi.Context) error {
    return c.ErrorUnauthorized("Custom unauthorized payload")
}
```

### Token Generation

```go
token, err := okapi.GenerateJwtToken([]byte(secret), jwt.MapClaims{
    "sub":  "123",
    "role": "admin",
}, 24*time.Hour)
```

### JWKS Loading

```go
jwks, err := okapi.LoadJWKSFromFile("path/to/jwks.json") // also accepts a base64-encoded JWKS
jwtAuth.JwksFile = jwks

// Or discover keys remotely
jwtAuth.JwksUrl = "https://issuer.example.com/.well-known/jwks.json"
```

`Jwks` holds `Keys []Jwk`; `Jwk` carries `Kid`, `Kty`, `N`/`E` (RSA) and `Crv`/`X`/`Y` (EC).

### Basic Authentication

```go
basicAuth := okapi.BasicAuth{
    Username:   "admin",
    Password:   "secret",
    Realm:      "Admin Area",
    ContextKey: "user", // where the username is stored; default "username"
}

o.Use(basicAuth.Middleware)                              // global
admin := o.Group("/admin", basicAuth.Middleware).WithBasicAuth() // group + docs
o.Get("/dashboard", h).Use(basicAuth.Middleware)         // per route
```

Credentials are compared in constant time; failures return 401 with a `WWW-Authenticate` header. (`BasicAuthMiddleware` is a deprecated alias of `BasicAuth`.)

### CORS

```go
o.WithCORS(okapi.Cors{
    AllowedOrigins:   []string{"https://app.example.com", "https://*.example.com"},
    AllowedHeaders:   []string{"Content-Type", "Authorization"},
    AllowMethods:     []string{"GET", "POST", "PUT", "DELETE"},
    ExposeHeaders:    []string{"X-Request-ID"},
    Headers:          map[string]string{"X-Frame-Options": "DENY"}, // extra response headers
    MaxAge:           3600,  // preflight cache seconds; <= 0 omits the header
    AllowCredentials: true,
})

// As an option at construction
o := okapi.New(okapi.WithCors(corsConfig))
```

Origin matching:

- `"*"` — any origin. With `AllowCredentials: true` the request origin is echoed verbatim so credentialed requests still work.
- `"https://*.example.com"` — scheme + wildcard subdomain.
- Exact origins are matched case-insensitively.

Empty `AllowedHeaders` / `AllowMethods` echo back `Access-Control-Request-Headers` / `Access-Control-Request-Method` on preflight.

Real preflights (OPTIONS **with** `Access-Control-Request-Method`) are short-circuited with 204; plain OPTIONS requests fall through to your handler. The middleware can also be attached directly as `corsConfig.CORSHandler`.
