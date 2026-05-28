## Okapi Overview

> A modern, minimalist HTTP framework for Go built for simplicity, performance, and developer experience.
> Built on `gorilla/mux` with `net/http` compatibility. Module: `github.com/jkaninda/okapi`

### Project Structure

```
okapi.go          - Core framework (Okapi struct, constructors, server lifecycle, route registration)
context.go        - Request/response context (data store, binding, responses, SSE, errors)
route.go          - Route and RouteDefinition types, route options
group.go          - Route groups with prefix, middleware, security, tags
binder.go         - Request binding (JSON, XML, YAML, Protobuf, form, multipart, query, header, path, cookie)
validator.go      - Struct tag validation (min, max, pattern, enum, format, const, etc.)
openapi.go        - OpenAPI 3.0/3.1 spec generation, doc options, DocBuilder, webhooks
doc.go            - Swagger UI / ReDoc / Scalar HTML templates and endpoints
middlewares.go    - Built-in middleware (Logger, BasicAuth, JWTAuth, BodyLimit, RequestID)
jwt.go            - JWT token generation, validation, key resolution
jwks.go           - JWKS loading (remote URL, file, base64)
jwt_claims_expression.go - DSL for JWT claim validation (Equals, Contains, OneOf, Prefix; &&, ||, !)
cors.go           - CORS handler configuration (origins, wildcards, credentials)
sse.go            - Server-Sent Events (Message, streaming, serializers)
template.go       - HTML template loading (files, directory, embedded FS, config)
renderer.go       - Renderer interface and RendererFunc adapter
errors.go         - Error types (ErrorResponse, ValidationError, ProblemDetail RFC 7807)
static.go         - Static file serving with directory listing prevention
tests.go          - TestServer and NewTestContext for unit tests
util.go           - Utility functions (TLS config loading, helpers)
helper.go         - Internal utilities
constants.go      - Default values, tag names, format types
var.go            - Package-level variables
version.go        - Version constant
client/           - Fluent HTTP client (retries, middleware, decoders, JSON/XML/YAML/multipart)
okapitest/        - Fluent HTTP test client (GET, POST, ... + assertions, status/body/header/JSON path)
okapicli/         - CLI integration (flags, env config, subcommands, struct-based config, lifecycle hooks)
examples/         - Example applications (sample, group, middleware, tls, sse, template, cli, client, tests, ...)
```

### Core Types

| Type | Description |
|------|-------------|
| `Okapi` | Main application struct |
| `Context` (aliases: `C`, `Ctx`) | Per-request context with store, binding, response helpers |
| `Route` | Registered route with metadata |
| `RouteDefinition` | Declarative route definition struct |
| `Group` | Route group with shared prefix, middleware, security, tags |
| `HandlerFunc` | `func(*Context) error` |
| `Middleware` | `func(*Context) error` — call `c.Next()` inside |
| `RouteOption` | `func(*Route)` - composable route configuration |
| `OptionFunc` | `func(*Okapi)` - composable app configuration |
| `M` | `map[string]any` shorthand |
| `ResponseWriter` | Extended `http.ResponseWriter` with status tracking, byte counting, hijack, flush, push |
| `Renderer` | Interface: `Render(io.Writer, string, interface{}, *Context) error` |
| `RendererFunc` | Function adapter implementing `Renderer` |
| `ErrorHandler` | `func(*Context, int, string, error) error` |
| `ErrorHandlerConfig` | RFC 7807 Problem Details configuration |
| `Cors` | CORS configuration struct |
| `BasicAuth` / `JWTAuth` / `BodyLimit` | Built-in middleware structs |
| `OpenAPI` | OpenAPI document configuration |
| `DocUI` | UI selector (`SwaggerUI`, `RedocUI`, `ScalarUI`) |

### Constructors

- `okapi.New(options ...OptionFunc) *Okapi` — Minimal instance (no docs by default)
- `okapi.Default() *Okapi` — With logger middleware and OpenAPI docs enabled

### App Configuration (OptionFunc / Chainable)

Most options are available as both `OptionFunc` (for `New()`) and chainable methods on `*Okapi`:

| Method | Description |
|--------|-------------|
| `WithPort(port int)` | Server port (default: 8080) |
| `WithAddr(addr string)` | Server address |
| `WithTLS(tlsConfig *tls.Config)` | Enable TLS |
| `WithTLSServer(addr string, tlsConfig *tls.Config)` | Separate TLS server alongside HTTP |
| `WithCors(cors Cors)` / `WithCORS(cors Cors)` | CORS configuration |
| `WithLogger(logger *slog.Logger)` | Structured logger |
| `WithContext(ctx context.Context)` | Application context |
| `WithDebug()` | Debug mode |
| `WithAccessLogDisabled()` / `DisableAccessLog()` | Disable access logging |
| `WithWriteTimeout(seconds int)` | HTTP write timeout |
| `WithReadTimeout(seconds int)` | HTTP read timeout |
| `WithIdleTimeout(seconds int)` | HTTP idle timeout |
| `WithStrictSlash(strict bool)` | Trailing slash behavior |
| `WithMaxMultipartMemory(max int64)` | Multipart memory limit (default: 32MB) |
| `WithMuxRouter(router *mux.Router)` | Custom gorilla/mux router |
| `WithServer(server *http.Server)` | Custom HTTP server |
| `WithOpenAPIDocs(cfg ...OpenAPI)` | Enable/configure OpenAPI docs |
| `WithOpenAPIDisabled()` | Disable OpenAPI docs |
| `WithDocUI(ui DocUI)` | Pick `SwaggerUI` / `RedocUI` / `ScalarUI` for `/docs` |
| `WithRenderer(renderer Renderer)` | Set template renderer |
| `WithRendererFromFS(fsys fs.FS, pattern string)` | Load from embedded FS |
| `WithRendererFromDirectory(dir string, ext ...string)` | Load from directory |
| `WithErrorHandler(handler ErrorHandler)` | Custom error handler |
| `WithDefaultErrorHandler()` | Reset to default error handler |
| `WithProblemDetailErrorHandler(config *ErrorHandlerConfig)` | RFC 7807 errors |
| `WithSimpleProblemDetailErrorHandler()` | RFC 7807 with defaults |

### Server Lifecycle

- `Start() error` — Start on configured port
- `StartOn(port int) error` — Start on specific port
- `StartServer(server *http.Server) error` — Start custom server
- `Stop() error` — Graceful shutdown
- `StopWithContext(ctx context.Context) error` — Shutdown with context
- `Shutdown(server *http.Server, ctx ...context.Context) error` — Shutdown specific server
- `GetContext() *Context` / `SetContext(*Context)` — Application context access

### Route Registration

- `o.Get/Post/Put/Delete/Patch/Head/Options/Any(path, handler, ...opts) *Route`
- `o.Handle(method, path, handler, ...opts)` — generic registration
- `o.HandleStd(method, path, http.HandlerFunc, ...opts)` — std-lib handler
- `o.HandleHTTP(method, path, http.Handler, ...opts)` — `http.Handler`
- `o.Register(routes ...RouteDefinition)` — declarative bulk
- `o.Group(prefix, ...middleware) *Group` — route group
- `o.Static(prefix, dir)` / `StaticFile(path, file)` / `StaticFS(prefix, http.FileSystem)`
- `o.NoRoute(handler)` / `o.NoMethod(handler)` — fallback handlers
- `o.Use(...Middleware)` — global Okapi middleware
- `o.UseMiddleware(func(http.Handler) http.Handler)` — bridge std-lib middleware
- `o.Webhook(name, method, ...opts) *Route` — declare OpenAPI 3.1 webhook (docs-only, no router entry)
- `o.Routes() []Route` — introspection
