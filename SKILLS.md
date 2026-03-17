# Okapi Skills Index

> AI Agent skills for the Okapi HTTP web framework for Go.
> Each skill is a focused, single-purpose reference in its own directory.

| Skill | Directory | Description |
|-------|-----------|-------------|
| Overview | `overview/` | Project structure, core types, constructors, app configuration, server lifecycle |
| Routing | `routing/` | HTTP methods, path parameters, generic handlers, route groups, route methods |
| Route Definition | `route_definition/` | Declarative `RouteDefinition` struct, bulk registration, project organization patterns |
| Request Binding | `request_binding/` | Struct tag binding (JSON, query, path, header, cookie, form), validation tags |
| Response | `response/` | JSON/XML/YAML responses, file serving, structured responses, error handling, RFC 7807 |
| OpenAPI | `openapi/` | Swagger UI, ReDoc, route documentation options, DocBuilder, OpenAPI configuration |
| Authentication | `authentication/` | JWT auth, claims expression DSL, Basic auth, CORS configuration |
| Middleware | `middleware/` | Built-in middleware, global/per-route/per-group middleware, chaining pattern |
| SSE Stream | `sse_stream/` | Server-Sent Events, single events, channel streaming, serializers |
| Testing | `testing/` | TestServer, TestContext, okapitest fluent client, assertions |
| CLI | `cli/` | okapicli package, flags, struct-based config, subcommands, server lifecycle |
| Context | `context/` | Data store, request inspection, parameters, cookies, templates, static files, TLS |
