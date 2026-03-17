# Okapi Skills

AI Agent skills for [Okapi](https://github.com/jkaninda/okapi) — a modern, minimalist HTTP web framework for Go inspired by FastAPI.

## What are Skills?

Skills are focused, single-purpose reference files that AI agents (Claude Code, Cursor, Copilot, etc.) can load to understand and work with the Okapi framework. Each skill lives in its own directory with a `SKILL.md` file containing the API surface, patterns, and examples for one specific topic.

## Skills

| Skill | Description |
|-------|-------------|
| [overview](overview/) | Project structure, core types, constructors, app configuration, server lifecycle |
| [routing](routing/) | HTTP methods, path parameters, generic handlers, route groups, route methods |
| [route_definition](route_definition/) | Declarative `RouteDefinition` struct, bulk registration, project organization patterns |
| [request_binding](request_binding/) | Struct tag binding (JSON, query, path, header, cookie, form), validation tags |
| [response](response/) | JSON/XML/YAML responses, file serving, structured responses, error handling, RFC 7807 |
| [openapi](openapi/) | Swagger UI, ReDoc, route documentation options, DocBuilder, OpenAPI configuration |
| [authentication](authentication/) | JWT auth, claims expression DSL, Basic auth, CORS configuration |
| [middleware](middleware/) | Built-in middleware, global/per-route/per-group middleware, chaining pattern |
| [sse_stream](sse_stream/) | Server-Sent Events, single events, channel streaming, serializers |
| [testing](testing/) | TestServer, TestContext, okapitest fluent client, assertions |
| [cli](cli/) | okapicli package, flags, struct-based config, subcommands, server lifecycle |
| [context](context/) | Data store, request inspection, parameters, cookies, templates, static files, TLS |

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
├── response/SKILL.md
├── openapi/SKILL.md
├── authentication/SKILL.md
├── middleware/SKILL.md
├── sse_stream/SKILL.md
├── testing/SKILL.md
├── cli/SKILL.md
└── context/SKILL.md
```

## Contributing

To add a new skill:

1. Create a directory with a descriptive name (e.g. `websocket/`)
2. Add a `SKILL.md` inside it with the API reference, types, and examples
3. Update `SKILLS.md` with the new entry

Keep skills focused on a single topic. Prefer code examples over prose.

## License

MIT — see the [Okapi repository](https://github.com/jkaninda/okapi) for details.
