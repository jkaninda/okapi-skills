## Okapi Authentication & CORS

### JWT Authentication

```go
jwtAuth := okapi.JWTAuth{
    SigningSecret:    []byte("secret"),          // HMAC secret
    RsaKey:          &publicKey,                 // OR RSA public key
    JwksUrl:         "https://.../.well-known/jwks.json", // OR remote JWKS
    JwksFile:        &okapi.Jwks{...},           // OR static JWKS
    Algo:            "RS256",                    // Expected algorithm
    Audience:        "my-api",                   // Expected audience
    Issuer:          "https://auth.example.com", // Expected issuer
    TokenLookup:     "header:Authorization",     // Or "query:token", "cookie:jwt"
    ContextKey:      "user",                     // Store claims in context
    ForwardClaims:   map[string]string{"email": "email", "role": "realm_access.roles"},
    ClaimsExpression: `And(Equals("email_verified","true"), OneOf("role","admin","editor"))`,
    ValidateClaims:  func(c *Context, claims jwt.Claims) error { ... },
    OnUnauthorized:  func(c *Context) error { ... },
}
protected := app.Group("/api", jwtAuth.Middleware).WithBearerAuth()
```

### Claims Expression DSL

```go
Equals(claimKey, expected)
Prefix(claimKey, prefix)
Contains(claimKey, ...values)
OneOf(claimKey, ...values)
And(left, right)
Or(left, right)
Not(expr)
```

### JWT Token Generation

```go
token, err := okapi.GenerateJwtToken(secret, jwt.MapClaims{"sub": "123"}, 24*time.Hour)
```

### JWKS Loading

```go
jwks, err := okapi.LoadJWKSFromFile("path/to/jwks.json")  // Or base64 string
```

### Basic Authentication

```go
basicAuth := okapi.BasicAuth{
    Username:   "admin",
    Password:   "password",
    Realm:      "Admin Area",
    ContextKey: "user",
}
admin := app.Group("/admin", basicAuth.Middleware)
```

### CORS

```go
app.WithCORS(okapi.Cors{
    AllowedOrigins:   []string{"https://example.com"},
    AllowedHeaders:   []string{"Content-Type", "Authorization"},
    AllowMethods:     []string{"GET", "POST", "PUT", "DELETE"},
    ExposeHeaders:    []string{"X-Request-ID"},
    MaxAge:           3600,
    AllowCredentials: true,
})
```
