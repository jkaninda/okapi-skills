## Okapi Middleware

### Built-in

- `okapi.LoggerMiddleware` - Access logging
- `okapi.BasicAuth{}.Middleware` - Basic authentication
- `okapi.JWTAuth{}.Middleware` - JWT authentication
- `okapi.BodyLimit{MaxBytes: 1<<20}.Middleware` - Request body size limit
- `okapi.Cors{}.CORSHandler` - CORS handling

### Global Middleware

```go
app.Use(middleware1, middleware2)
app.UseMiddleware(stdHttpMiddleware)  // func(http.Handler) http.Handler
```

### Per-Route / Per-Group Middleware

```go
route.Use(cacheMiddleware)
group.Use(authMiddleware)
okapi.UseMiddleware(middleware)  // As RouteOption
```

### Middleware Chaining

```go
// Context-based middleware with Next()
func myMiddleware(next okapi.HandlerFunc) okapi.HandlerFunc {
    return func(c *okapi.Context) error {
        // before
        err := c.Next()
        // after
        return err
    }
}
```
