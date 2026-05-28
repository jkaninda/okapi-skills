# Okapi Skills Index

> AI Agent skills for the Okapi HTTP web framework for Go.
> Each skill is a focused, single-purpose reference in its own directory.

| Skill | Directory | Description |
|-------|-----------|-------------|
| Overview | `overview/` | Project structure, core types, constructors, app configuration, server lifecycle |
| Routing | `routing/` | HTTP methods, path parameters, generic handlers, route groups, route methods, fallbacks |
| Route Definition | `route_definition/` | Declarative `RouteDefinition` struct, bulk registration, project organization patterns |
| Request Binding | `request_binding/` | Struct tag binding (JSON, query, path, header, cookie, form), validation tags, formats |
| Response | `response/` | JSON/XML/YAML responses, file serving, structured responses, ResponseWriter extensions |
| Error Handling | `error_handling/` | Abort methods, custom error handlers, RFC 7807 Problem Details |
| OpenAPI | `openapi/` | Swagger UI, ReDoc, Scalar, OpenAPI 3.0 / 3.1, webhooks, OAuth flows, DocBuilder |
| Authentication | `authentication/` | JWT auth, claims expression DSL, Basic auth, CORS configuration |
| Middleware | `middleware/` | Built-in middleware (Logger, RequestID, BasicAuth, JWT, BodyLimit, CORS), chaining, std-lib bridge |
| Dynamic Routes | `dynamic_routes/` | Runtime enable/disable of routes, groups, and docs; deprecation; hiding |
| SSE Stream | `sse_stream/` | Server-Sent Events, single events, channel streaming, serializers |
| WebSocket | `websocket/` | WebSocket upgrade via the `okapi-ws` package |
| HTTP Client | `http_client/` | `okapi/client` fluent HTTP client, retries, middleware, decoders |
| Testing | `testing/` | TestServer, TestContext, okapitest fluent client, assertions |
| CLI | `cli/` | okapicli package, flags, struct-based config, subcommands, server lifecycle hooks |
| Context | `context/` | Data store, request inspection, parameters, cookies, templates, static files, TLS |
