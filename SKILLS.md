# Okapi Skills Index

> AI Agent skills for the [Okapi](https://github.com/jkaninda/okapi) HTTP web framework for Go.
> Each skill is a focused, single-purpose reference in its own directory.

| Skill | Directory | Description |
|-------|-----------|-------------|
| Overview | `overview/` | Project structure, core types, constructors, app configuration, server lifecycle |
| Routing | `routing/` | HTTP methods, path parameters, generic handlers, route groups, route methods, fallbacks |
| Route Definition | `route_definition/` | Declarative `RouteDefinition` struct, bulk registration, project organization patterns |
| Request Binding | `request_binding/` | Binding sources (JSON, XML, YAML, Protobuf, query, path, header, cookie, form), body-field style, uploads |
| Validation | `validation/` | Struct-tag constraints, conditional required, schema annotations, formats, typed handlers |
| Response | `response/` | JSON/XML/YAML responses, file serving, structured responses, write-once semantics, ResponseWriter |
| Error Handling | `error_handling/` | Abort and Error helpers, custom error handlers, RFC 7807 Problem Details, validation errors |
| OpenAPI | `openapi/` | Swagger UI, ReDoc, Scalar, OpenAPI 3.1/3.0, webhooks, OAuth flows, DocBuilder, component schemas |
| Authentication | `authentication/` | JWT (JWKS, claims expression DSL, claim forwarding), Basic auth, CORS |
| Middleware | `middleware/` | Built-in middleware, chaining, `c.Next()`, std-lib bridge, custom middleware |
| Dynamic Routes | `dynamic_routes/` | Runtime enable/disable of routes, groups, and docs; deprecation; hiding |
| SSE Stream | `sse_stream/` | Server-Sent Events, single events, channel streaming, serializers |
| WebSocket | `websocket/` | WebSocket server and client via the `okapiws` package |
| HTTP Client | `http_client/` | `okapi/client` fluent HTTP client, retries, middleware, decoders |
| Testing | `testing/` | TestServer, TestContext, okapitest fluent client, assertions |
| CLI | `cli/` | okapicli package, flags, struct-based config, subcommands, server lifecycle hooks |
| Context | `context/` | Data store, request inspection, parameters, cookies, goroutine safety |
| Templating | `templating/` | Template loading (files, directory, embedded FS), renderers, HTML helpers |
| Web / SPA | `web_spa/` | `Web` / `WebFS` single-page app serving with index fallback, static files |
| TLS & HTTPS | `tls_https/` | `LoadTLSConfig`, dual HTTP/HTTPS, mTLS, autocert, HSTS |
| Std-lib Compatibility | `std_compat/` | `net/http` handlers and middleware, path params, gradual migration |
