# Okapi Skills

AI Agent skills for [Okapi](https://github.com/jkaninda/okapi) — a modern, minimalist HTTP framework for Go built for simplicity, performance, and developer experience.

## What are Skills?

Skills are focused, single-purpose reference files that AI agents (Claude Code, Cursor, Copilot, etc.) can load to understand and work with the Okapi framework. Each skill lives in its own directory with a `SKILL.md` file containing the API surface, patterns, and examples for one specific topic.

## Skills

| Skill | Description |
|-------|-------------|
| [overview](overview/) | Project structure, core types, constructors, app configuration, server lifecycle |
| [routing](routing/) | HTTP methods, path parameters, generic handlers, route groups, route methods, fallbacks |
| [route_definition](route_definition/) | Declarative `RouteDefinition` struct, bulk registration, project organization patterns |
| [request_binding](request_binding/) | Binding sources (JSON, XML, YAML, Protobuf, query, path, header, cookie, form), body-field style, uploads |
| [validation](validation/) | Struct-tag constraints, conditional required, schema annotations, formats, typed handlers |
| [response](response/) | JSON/XML/YAML responses, file serving, structured responses, write-once semantics, ResponseWriter |
| [error_handling](error_handling/) | Abort and Error helpers, custom error handlers, RFC 7807 Problem Details, validation errors |
| [openapi](openapi/) | Swagger UI, ReDoc, Scalar, OpenAPI 3.1 / 3.0, webhooks, OAuth flows, DocBuilder, component schemas |
| [authentication](authentication/) | JWT (JWKS, claims expression DSL, claim forwarding), Basic auth, CORS |
| [middleware](middleware/) | Built-in middleware (Logger, RequestID, BasicAuth, JWT, BodyLimit, CORS), chaining, std-lib bridge |
| [dynamic_routes](dynamic_routes/) | Runtime enable/disable of routes, groups, and docs; deprecation; hiding |
| [sse_stream](sse_stream/) | Server-Sent Events, single events, channel streaming, serializers |
| [websocket](websocket/) | WebSocket server and client via the `okapiws` package, with Okapi and `net/http` |
| [http_client](http_client/) | `okapi/client` fluent HTTP client, retries, middleware, decoders |
| [testing](testing/) | TestServer, TestContext, okapitest fluent client, assertions |
| [cli](cli/) | okapicli package, flags, struct-based config, subcommands, server lifecycle hooks |
| [context](context/) | Data store, request inspection, parameters, cookies, goroutine safety |
| [templating](templating/) | Template loading (files, directory, embedded FS), renderers, HTML helpers |
| [web_spa](web_spa/) | `Web` / `WebFS` single-page app serving with index fallback, static files |
| [tls_https](tls_https/) | `LoadTLSConfig`, dual HTTP/HTTPS, mTLS, autocert, HSTS |
| [std_compat](std_compat/) | `net/http` handlers and middleware, path parameters, gradual migration |

## Usage

### Claude Code

Add skills to your project by referencing them in your `CLAUDE.md`:

```markdown
Read skills from https://github.com/jkaninda/okapi-skills
```

Or clone locally and point to specific skills:

```bash
git clone https://github.com/jkaninda/okapi-skills.git skills/okapi
```

### Other AI Agents

Copy the relevant `SKILL.md` files into your project's context or documentation directory. Each file is self-contained and can be loaded independently.

## Structure

```
skills/
├── README.md              # This file
├── SKILLS.md              # Machine-readable index
├── overview/SKILL.md
├── routing/SKILL.md
├── route_definition/SKILL.md
├── request_binding/SKILL.md
├── validation/SKILL.md
├── response/SKILL.md
├── error_handling/SKILL.md
├── openapi/SKILL.md
├── authentication/SKILL.md
├── middleware/SKILL.md
├── dynamic_routes/SKILL.md
├── sse_stream/SKILL.md
├── websocket/SKILL.md
├── http_client/SKILL.md
├── testing/SKILL.md
├── cli/SKILL.md
├── context/SKILL.md
├── templating/SKILL.md
├── web_spa/SKILL.md
├── tls_https/SKILL.md
└── std_compat/SKILL.md
```

## Contributing

To add a new skill:

1. Create a directory with a descriptive name (e.g. `metrics/`)
2. Add a `SKILL.md` inside it with the API reference, types, and examples
3. Update `README.md` and `SKILLS.md` with the new entry

Keep skills focused on a single topic. Prefer code examples over prose.

## License

MIT — see the [Okapi repository](https://github.com/jkaninda/okapi) for details.
