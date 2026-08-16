## Okapi Overview

> A modern, minimalist HTTP framework for Go built for simplicity, performance, and developer experience.
> Module: `github.com/jkaninda/okapi` — requires Go 1.25+. Implements `http.Handler`, fully `net/http` compatible.
> Routing is provided by `github.com/jkaninda/njia` (Okapi's own router; it replaced the archived `gorilla/mux` in v0.10.0).

```bash
go get github.com/jkaninda/okapi
```

### Project Structure

```
okapi.go          - Core framework (Okapi struct, options, server lifecycle, route registration)
router_register.go- Router registration helpers
context.go        - Request/response context (data store, binding, responses, SSE, errors)
route.go          - Route and RouteDefinition types, route options
group.go          - Route groups with prefix, middleware, security, tags
binder.go         - Request binding (JSON, XML, YAML, Protobuf, form, multipart, query, header, path, cookie)
validator.go      - Struct tag validation (min/max, length, pattern, enum, format, const, conditional required)
openapi.go        - OpenAPI 3.0/3.1 spec generation, doc options, DocBuilder, webhooks
doc.go            - Swagger UI / ReDoc / Scalar HTML templates and endpoints
middlewares.go    - Built-in middleware (Logger, BasicAuth, JWTAuth, BodyLimit, RequestID)
jwt.go            - JWT token generation, validation, key resolution
jwks.go           - JWKS loading (remote URL, file, base64)
jwt_claims_expression.go - DSL for JWT claim validation (Equals, Contains, OneOf, Prefix; &&, ||, !)
cors.go           - CORS handler configuration (origins, wildcards, credentials)
sse.go            - Server-Sent Events (Message, StreamOptions, serializers)
template.go       - HTML template loading (files, directory, embedded FS, config)
renderer.go       - Renderer interface and RendererFunc adapter
errors.go         - Error types (ErrorResponse, ValidationError, ProblemDetail RFC 7807), Abort/Error helpers
static.go         - Static files and Web/WebFS SPA serving with index fallback
tests.go          - TestServer and NewTestContext for tests
util.go           - Utilities (LoadTLSConfig, ValidateAddr, status-class helpers)
helper.go         - Internal helpers
constants.go      - Default values, tag names, format types
var.go            - Package-level variables
client/           - Fluent HTTP client (retries, middleware, decoders, JSON/XML/YAML/multipart)
okapitest/        - Fluent HTTP test client (verbs + status/body/header/JSON-path assertions)
okapicli/         - CLI integration (flags, env config, subcommands, struct config, lifecycle hooks)
examples/         - Runnable examples (sample, group, middleware, tls, sse, template, web, cli, client, tests, std, ...)
```

### Core Types

| Type | Description |
|------|-------------|
| `Okapi` | Main application struct — **all fields unexported**; configure via options/methods |
| `Context` (aliases: `C`, `Ctx`) | Per-request context with store, binding, response helpers |
| `Route` | Registered route with metadata |
| `RouteDefinition` | Declarative route definition struct |
| `Group` | Route group with shared prefix, middleware, security, tags |
| `GroupTag` | Named OpenAPI tag with description + external docs |
| `HandlerFunc` | `func(*Context) error` |
| `Middleware` / `MiddlewareFunc` | Type aliases of `HandlerFunc` — call `c.Next()` inside |
| `RouteOption` | `func(*Route)` — composable route configuration |
| `OptionFunc` | `func(*Okapi)` — composable app configuration |
| `M` | `map[string]any` shorthand |
| `ResponseWriter` | Extended `http.ResponseWriter` (status, byte count, hijack, flush, push, close) |
| `Renderer` / `RendererFunc` | Template rendering interface + function adapter |
| `Template` / `TemplateConfig` | Built-in `html/template` loader |
| `ErrorHandler` | `func(*Context, int, string, error) error` |
| `ErrorHandlerConfig` / `ProblemDetail` | RFC 7807 configuration and payload |
| `ErrorResponse` / `ValidationError` / `ValidationErrorResponse` | Default error payloads |
| `Cors` | CORS configuration struct |
| `BasicAuth` / `JWTAuth` / `BodyLimit` | Built-in middleware structs |
| `Jwks` / `Jwk` | JWKS key material |
| `OpenAPI` | OpenAPI document configuration |
| `DocUI` | UI selector (`SwaggerUI`, `RedocUI`, `ScalarUI`) |
| `DocBuilder` | Fluent route documentation builder (`okapi.Doc()`) |
| `SchemaInfo` | Reusable component schema registration |
| `Message` / `StreamOptions` / `Serializer` | SSE primitives |
| `WebConfig` | SPA / web app serving configuration |
| `TestServer` / `TestingT` | Test server helpers |

### Constructors

- `okapi.New(options ...OptionFunc) *Okapi` — minimal instance (no docs, no access log)
- `okapi.Default() *Okapi` — logger middleware + access log + OpenAPI docs enabled
- `okapi.NewGroup(basePath string, o *Okapi, middlewares ...Middleware) *Group`
- `okapi.NewContext(o *Okapi, w http.ResponseWriter, r *http.Request) *Context`

### App Configuration

Most options exist both as an `OptionFunc` (for `New()` / `With()`) and as a chainable method on `*Okapi`.

| Option | Description |
|--------|-------------|
| `WithPort(port int)` | Server port (default 8080) |
| `WithAddr(addr string)` | Server address (default `:8080`) |
| `WithServer(*http.Server)` | Supply a fully configured `*http.Server` (option only) |
| `WithTLS(*tls.Config)` | Serve HTTPS on the main server |
| `WithTLSServer(addr string, *tls.Config)` | Additional HTTPS listener alongside HTTP (option only) |
| `WithCors(cors Cors)` (option) / `WithCORS(cors Cors)` (method) | CORS configuration |
| `WithLogger(*slog.Logger)` | Structured logger |
| `WithContext(context.Context)` | Application context |
| `WithDebug()` | Debug mode + access logging |
| `WithAccessLogDisabled()` (option) / `DisableAccessLog()` (method) | Disable access logging |
| `WithReadTimeout(s int)` / `WithWriteTimeout(s int)` / `WithIdleTimeout(s int)` | Server timeouts (seconds) |
| `WithStrictSlash(strict bool)` | Trailing-slash redirect behaviour |
| `WithMaxMultipartMemory(max int64)` | Multipart memory limit (default 32 MB) |
| `WithOpenAPIDocs(cfg ...OpenAPI)` | Enable/configure OpenAPI docs |
| `WithOpenAPIDisabled()` | Disable OpenAPI docs |
| `WithDocUI(ui DocUI)` | Pick `SwaggerUI` / `RedocUI` / `ScalarUI` for `/docs` |
| `WithRenderer(Renderer)` | Set the template renderer |
| `WithDefaultRenderer(pattern string)` (method) | Renderer from a file pattern |
| `WithRendererFromFS(fsys fs.FS, pattern string)` (method) | Renderer from an embedded FS |
| `WithRendererFromDirectory(dir string, ext ...string)` (method) | Renderer from a directory |
| `WithRendererConfig(TemplateConfig)` (method) | Renderer from a config struct |
| `WithErrorHandler(ErrorHandler)` | Custom error handler |
| `WithDefaultErrorHandler()` | Reset to the default error handler |
| `WithProblemDetailErrorHandler(*ErrorHandlerConfig)` | RFC 7807 errors |
| `WithSimpleProblemDetailErrorHandler()` | RFC 7807 with defaults |
| `WithMuxRouter(*njia.Router)` | **Deprecated** no-op — Okapi manages its own router |

`o.With(options ...OptionFunc) *Okapi` applies options to an existing instance.

### Server Lifecycle

```go
o.Start() error                                  // start on the configured addr (HTTPS if TLSConfig set)
o.StartOn(port int) error                        // start on a specific port
o.StartServer(server *http.Server) error         // start with a supplied server
o.Stop() error                                   // graceful shutdown of HTTP + HTTPS listeners
o.StopWithContext(ctx context.Context) error     // graceful shutdown with a deadline
o.Shutdown(server *http.Server, ctx ...context.Context) error // deprecated — use StopWithContext
o.WaitForServer(timeout time.Duration) string    // block until ready, return the address
o.StartForTest(t TestingT) string                // start for a test, return the base URL
o.ServeHTTP(w, r)                                // Okapi is an http.Handler
o.GetContext() *Context / o.SetContext(*Context)
```

Graceful shutdown with signals is handled for you by `okapicli` (`cli.Run()` / `cli.RunServer(...)`) — see the `cli/` skill.

### Route Registration

```go
o.Get/Post/Put/Delete/Patch/Head/Options/Any(path, handler, ...opts) *Route
o.Handle(method, path, handler, ...opts)                    // generic registration
o.HandleStd(method, path, http.HandlerFunc, ...opts)        // std-lib handler
o.HandleHTTP(method, path, http.Handler, ...opts)           // http.Handler
o.Register(routes ...RouteDefinition)                       // declarative bulk registration
okapi.RegisterRoutes(o, []RouteDefinition)                  // package-level equivalent
o.Group(prefix, ...middleware) *Group                       // route group
o.Use(...Middleware)                                        // global middleware
o.UseMiddleware(func(http.Handler) http.Handler)            // std-lib middleware bridge
o.NoRoute(handler) / o.NoMethod(handler)                    // 404 / 405 fallbacks
o.Webhook(name, method, ...opts) *Route                     // OpenAPI 3.1 webhook (docs-only)
o.RegisterSchemas(map[string]*SchemaInfo) error             // reusable component schemas
o.Routes() []Route                                          // introspection snapshot
```

### Static Files & Web Apps

```go
o.Static(prefix, dir)                    // serve a directory (no listing)
o.StaticFile(path, filepath)             // serve a single file
o.StaticFS(prefix, http.FileSystem)      // serve from any http.FileSystem
o.Web(prefix, dir, ...WebConfig)         // SPA from disk, index fallback
o.WebFS(prefix, fs.FS, ...WebConfig)     // SPA from an fs.FS / embed.FS
o.SPA(...) / o.SPAFS(...)                // deprecated aliases of Web / WebFS
```

### Package-Level Helpers

```go
okapi.GenerateJwtToken(secret []byte, claims jwt.MapClaims, ttl time.Duration) (string, error)
okapi.LoadJWKSFromFile(pathOrBase64 string) (*Jwks, error)
okapi.LoadTLSConfig(certFile, keyFile, caFile string, clientAuth bool) (*tls.Config, error)
okapi.ValidateAddr(addr string) bool
okapi.IsError(code int) bool / IsClientError(code int) bool / IsServerError(code int) bool
okapi.DefaultErrorHandler(c, code, message, err) error
okapi.ProblemDetailErrorHandler(cfg *ErrorHandlerConfig) ErrorHandler
```

### Minimal Application

```go
package main

import "github.com/jkaninda/okapi"

func main() {
    o := okapi.Default() // logger + OpenAPI docs at /docs

    o.Get("/", func(c *okapi.Context) error {
        return c.OK(okapi.M{"message": "Hello from Okapi!"})
    })

    if err := o.Start(); err != nil {
        panic(err)
    }
}
```

### Skill Map

| Topic | Skill |
|-------|-------|
| Methods, paths, groups, fallbacks | `routing/` |
| Declarative route structs | `route_definition/` |
| Binding sources and methods | `request_binding/` |
| Struct-tag validation and formats | `validation/` |
| Writing responses | `response/` |
| Aborts, error handlers, RFC 7807 | `error_handling/` |
| Specs, UIs, doc options, webhooks | `openapi/` |
| JWT, Basic auth, CORS | `authentication/` |
| Built-in and custom middleware | `middleware/` |
| Runtime enable/disable | `dynamic_routes/` |
| Server-Sent Events | `sse_stream/` |
| WebSockets (`okapiws`) | `websocket/` |
| Outbound HTTP client | `http_client/` |
| Test server and assertions | `testing/` |
| Flags, subcommands, lifecycle | `cli/` |
| Context utilities | `context/` |
| Templates and renderers | `templating/` |
| SPA / static serving | `web_spa/` |
| TLS, mTLS, autocert | `tls_https/` |
| `net/http` interop | `std_compat/` |
