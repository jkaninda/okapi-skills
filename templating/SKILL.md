## Okapi Templating & Rendering

Okapi renders HTML through a `Renderer`. The built-in `Template` type wraps `html/template` and loads from a directory, a glob pattern, an embedded FS, or a config struct.

### Renderer Interface

```go
type Renderer interface {
    Render(w io.Writer, name string, data interface{}, c *Context) error
}

type RendererFunc func(io.Writer, string, interface{}, *Context) error // adapter
```

### Loading Templates

```go
// 1. Directory scan (with extension filter)
tmpl, err := okapi.NewTemplateFromDirectory("public/views", ".html", ".tmpl")
o.WithRenderer(tmpl)

// 2. Glob pattern
tmpl, err := okapi.NewTemplateFromFiles("public/views/*.html")
o.WithRenderer(tmpl)

// 3. Embedded FS (recommended for production)
//go:embed views/*
var views embed.FS
tmpl, err := okapi.NewTemplate(views, "views/*.html")
o.WithRenderer(tmpl)

// 4. Config struct (custom funcs, custom FS)
tmpl, err := okapi.NewTemplateWithConfig(okapi.TemplateConfig{
    FS:      os.DirFS("templates"),
    Pattern: "**/*.html",
    BaseDir: "templates",
    Funcs:   template.FuncMap{"upper": strings.ToUpper},
})
o.WithRenderer(tmpl)
```

### Shortcuts on `*Okapi`

Each avoids constructing the `*Template` yourself:

```go
o.WithDefaultRenderer("public/views/*.html")            // from a file pattern
o.WithRendererFromDirectory("public/views", ".html")    // from a directory
o.WithRendererFromFS(views, "views/*.html")             // from an fs.FS
o.WithRendererConfig(okapi.TemplateConfig{...})         // from a config struct
o.WithRenderer(customRenderer)                          // any Renderer implementation
```

All are also available as `OptionFunc` where noted (`okapi.WithRenderer(...)`).

### TemplateConfig

| Field | Type | Description |
|-------|------|-------------|
| `Pattern` | `string` | File pattern (e.g. `views/*.html`) |
| `FS` | `fs.FS` | Embedded or custom filesystem |
| `Funcs` | `template.FuncMap` | Custom template functions |
| `BaseDir` | `string` | Base directory for templates |

### Adding Templates at Runtime

```go
err := tmpl.AddTemplate("banner", "<h1>{{.Title}}</h1>") // from a string
err = tmpl.AddTemplateFile("path/to/template.html")      // from a file
```

### Rendering in a Handler

```go
o.Get("/", func(c *okapi.Context) error {
    return c.Render(http.StatusOK, "home", okapi.M{
        "title": "Welcome",
        "user":  user,
    })
})
```

`okapi.M` is shorthand for `map[string]any`; each key is accessible in the template as `{{.title}}`.

Other HTML helpers on `*Context`:

```go
c.Render(code, name, data)          // named template via the configured Renderer
c.HTML(code, file, data)            // parse and render a template file directly
c.HTMLView(code, templateStr, data) // render an inline template string
```

`c.Render` returns `okapi.ErrNoRenderer` when no renderer is configured.

### Custom Renderer

```go
// Function adapter — light, parses per request
o.WithRenderer(okapi.RendererFunc(func(w io.Writer, name string, data any, c *okapi.Context) error {
    tmpl, err := template.ParseFiles("templates/" + name + ".html")
    if err != nil {
        return err
    }
    return tmpl.ExecuteTemplate(w, name, data)
}))

// Struct-based — templates parsed once at startup (preferred in production)
type Template struct{ templates *template.Template }

func (t *Template) Render(w io.Writer, name string, data any, c *okapi.Context) error {
    return t.templates.ExecuteTemplate(w, name, data)
}

o.WithRenderer(&Template{
    templates: template.Must(template.ParseGlob("public/views/*.html")),
})
```

> The `*Okapi` struct has no exported fields — always install a renderer with `WithRenderer` (or one of the `WithRenderer*` helpers), never by assigning to a field.

### Content Negotiation Caveat

`c.Respond` / `c.Return` (and the `HandleIO` / `HandleO` wrappers) pick the format from the request's `Accept` header: XML, YAML, JSON, or — for `text/plain` **and `text/html`** — plain text via `c.String`. They do **not** route through the renderer. To return rendered HTML, call `c.Render` / `c.HTML` / `c.HTMLView` explicitly.
