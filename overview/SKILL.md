## Okapi Overview

> A modern, minimalist HTTP web framework for Go inspired by FastAPI's elegant design philosophy.
> Built on `gorilla/mux` with `net/http` compatibility. Module: `github.com/jkaninda/okapi`

### Project Structure

```
okapi.go          - Core framework (Okapi struct, constructors, server lifecycle, route registration)
context.go        - Request/response context (data store, binding, responses, SSE, errors)
route.go          - Route and RouteDefinition types, route options
group.go          - Route groups with prefix, middleware, security
binder.go         - Request binding (JSON, XML, YAML, Protobuf, form, multipart, query, header, path, cookie)
validator.go      - Struct tag validation (min, max, pattern, enum, format, etc.)
openapi.go        - OpenAPI spec generation, doc options, DocBuilder
doc.go            - Swagger UI / ReDoc HTML templates and endpoints
middlewares.go    - Built-in middleware (logger, BasicAuth, JWTAuth, BodyLimit)
jwt.go            - JWT token generation, validation, key resolution
jwks.go           - JWKS loading (remote URL, file, base64)
jwt_claims_expression.go - DSL for JWT claim validation (Equals, Contains, OneOf, And, Or, Not)
cors.go           - CORS middleware configuration
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
okapitest/        - Fluent HTTP test client (GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS + assertions)
okapicli/         - CLI integration (flags, env config, subcommands, struct-based config, signal handling)
examples/         - Example applications (sample, group, middleware, tls, sse, template, cli, etc.)
```

### Core Types

| Type | Description |
|------|-------------|
| `Okapi` | Main application struct |
| `Context` (aliases: `C`, `Ctx`) | Per-request context with store, binding, response helpers |
| `Route` | Registered route with metadata |
| `RouteDefinition` | Declarative route definition struct |
| `Group` | Route group with shared prefix, middleware, security |
| `HandlerFunc` | `func(*Context) error` |
| `Middleware` | `func(next HandlerFunc) HandlerFunc` |
| `RouteOption` | `func(*Route)` - composable route configuration |
| `OptionFunc` | `func(*Okapi)` - composable app configuration |
| `M` | `map[string]any` shorthand |
| `ResponseWriter` | Extended `http.ResponseWriter` with status tracking, byte counting, hijack, flush, push |
| `Renderer` | Interface: `Render(io.Writer, string, interface{}, *Context) error` |
| `ErrorHandler` | `func(*Context, int, string, error) error` |

### Constructors

- `okapi.New(options ...OptionFunc) *Okapi` - Minimal instance (no docs by default)
- `okapi.Default() *Okapi` - With logger middleware and OpenAPI docs enabled

### App Configuration (OptionFunc / Chainable)

All available as both `OptionFunc` (for `New()`) and chainable methods on `*Okapi`:

| Method | Description |
|--------|-------------|
| `WithPort(port int)` | Server port (default: 8080) |
| `WithAddr(addr string)` | Server address |
| `WithTLS(tlsConfig *tls.Config)` | Enable TLS |
| `WithTLSServer(addr string, tlsConfig *tls.Config)` | Separate TLS server |
| `WithCors(cors Cors)` / `WithCORS(cors Cors)` | CORS configuration |
| `WithLogger(logger *slog.Logger)` | Structured logger |
| `WithContext(ctx context.Context)` | Application context |
| `WithDebug()` | Debug mode |
| `DisableAccessLog()` / `WithAccessLogDisabled()` | Disable access logging |
| `WithWriteTimeout(seconds int)` | HTTP write timeout |
| `WithReadTimeout(seconds int)` | HTTP read timeout |
| `WithIdleTimeout(seconds int)` | HTTP idle timeout |
| `WithStrictSlash(strict bool)` | Trailing slash behavior |
| `WithMaxMultipartMemory(max int64)` | Multipart memory limit (default: 32MB) |
| `WithMuxRouter(router *mux.Router)` | Custom gorilla/mux router |
| `WithServer(server *http.Server)` | Custom HTTP server |
| `WithOpenAPIDocs(cfg ...OpenAPI)` | Enable/configure OpenAPI docs |
| `WithOpenAPIDisabled()` | Disable OpenAPI docs |
| `WithRenderer(renderer Renderer)` | Set template renderer |
| `WithDefaultRenderer(templatePath string)` | Load templates from path |
| `WithRendererFromFS(fsys fs.FS, pattern string)` | Load from embedded FS |
| `WithRendererFromDirectory(dir string, ext ...string)` | Load from directory |
| `WithRendererConfig(config TemplateConfig)` | Load with config |
| `WithErrorHandler(handler ErrorHandler)` | Custom error handler |
| `WithDefaultErrorHandler()` | Reset to default error handler |
| `WithProblemDetailErrorHandler(config *ErrorHandlerConfig)` | RFC 7807 errors |
| `WithSimpleProblemDetailErrorHandler()` | RFC 7807 with defaults |

### Server Lifecycle

- `Start() error` - Start on configured port
- `StartOn(port int) error` - Start on specific port
- `StartServer(server *http.Server) error` - Start custom server
- `Stop() error` - Graceful shutdown
- `StopWithContext(ctx context.Context) error` - Shutdown with context
- `Shutdown(server *http.Server, ctx ...context.Context) error` - Shutdown specific server
- `GetContext() context.Context` / `SetContext(ctx context.Context)` - Application context access
