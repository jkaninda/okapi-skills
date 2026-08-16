## Okapi CLI Integration (`okapicli` package)

The `okapicli` package adds typed flags, env-variable binding, subcommands, and lifecycle hooks to an Okapi app.

### Constructors

```go
cli := okapicli.New(app *okapi.Okapi, name ...string) *CLI // wrap an existing app
cli := okapicli.Default() *CLI                              // new CLI with a default Okapi instance
app := cli.Okapi()                                          // the underlying *okapi.Okapi
```

### Basic Usage

```go
import "github.com/jkaninda/okapi/okapicli"

cli := okapicli.New(app, "myapp").
    String("config", "c", "config.yaml", "Path to configuration file").
    Int("port", "p", 8080, "HTTP server port").
    Bool("debug", "d", false, "Enable debug mode").
    Float("rate", "r", 1.0, "Rate limit").
    Duration("timeout", "t", 30*time.Second, "Request timeout")

if err := cli.Parse(); err != nil { panic(err) }

app.WithPort(cli.GetInt("port")).WithDebug(cli.GetBool("debug"))

cli.Run()
```

Flag types: `String`, `Int`, `Bool`, `Float`, `Duration`. Each accepts `(name, shortName, default, description)`.

### Struct-Based Configuration

Tag your config struct and let `okapicli` register flags + bind env vars in one call.

| Tag | Description |
|-----|-------------|
| `cli` | Flag name (required) |
| `short` | Short flag name (optional) |
| `desc` | `--help` description |
| `default` | Default value (string, parsed to field type) |
| `env` | Environment variable name |

```go
type Config struct {
    Port    int           `cli:"port"    short:"p" desc:"Server port"    env:"APP_PORT"    default:"8080"`
    Host    string        `cli:"host"    short:"h" desc:"Hostname"        env:"APP_HOST"    default:"localhost"`
    Debug   bool          `cli:"debug"   short:"d" desc:"Debug mode"     env:"APP_DEBUG"`
    Config  string        `cli:"config"  short:"c" desc:"Config file"    env:"APP_CONFIG"  default:"config.yaml"`
    Timeout time.Duration `cli:"timeout" short:"t" desc:"Request timeout" env:"APP_TIMEOUT" default:"30s"`
}

cfg := &Config{Port: 8000} // struct defaults still win over the zero value but lose to all tags
cli := okapicli.New(app, "myapp").FromStruct(cfg)
if err := cli.Parse(); err != nil { panic(err) }
// cfg fields are populated with resolved values
```

`cli.WithConfig(cfg)` is an equivalent of `FromStruct` with the same tag set. Supported field types: `string`, `int*`, `bool`, `float*`.

### Value Resolution Order

Lowest → highest priority:

1. Struct field's initial value
2. `default` tag
3. Environment variable (`env`)
4. CLI flag

### One-Liner Parsing (Fail Fast)

```go
cli := okapicli.New(app, "myapp").FromStruct(cfg).MustParse() // panics on error
```

### Subcommands

```go
cli.Command("serve", "Start the HTTP server", func(cmd *okapicli.Command) error {
    cmd.Okapi().WithPort(cmd.GetInt("port"))
    return cmd.CLI().Run()
}).Int("port", "p", 8080, "HTTP server port")

cli.Command("migrate", "Run database migrations", func(cmd *okapicli.Command) error {
    return runMigrations(cmd.GetString("dsn"))
}).String("dsn", "", "", "Database connection string")

cli.DefaultCommand("serve") // run "serve" when no subcommand specified
cli.Execute()
```

Command methods:

```go
cmd.Name() string              // Command name
cmd.CLI() *CLI                 // Parent CLI instance
cmd.Okapi() *okapi.Okapi       // Okapi instance from parent CLI
cmd.Args() []string            // Non-flag arguments
cmd.GetString(name) string
cmd.GetInt(name) int
cmd.GetBool(name) bool
cmd.GetFloat(name) float64
cmd.GetDuration(name) time.Duration
cmd.FromStruct(v)              // Register flags from struct tags
```

### Server Lifecycle Hooks

```go
cli.RunServer(&okapicli.RunOptions{
    ShutdownTimeout: 30 * time.Second,
    Signals:         []os.Signal{okapicli.SIGINT, okapicli.SIGTERM},
    OnStart:    func() { slog.Info("Preparing resources before startup") },
    OnStarted:  func() { slog.Info("Server started") },
    OnShutdown: func() { slog.Info("Cleaning up before shutdown") },
})

// Or simple defaults — shorthand for RunServer(nil): 30s shutdown timeout, SIGINT/SIGTERM
cli.Run()
```

| `RunOptions` field | Default |
|--------------------|---------|
| `ShutdownTimeout` | 30s |
| `Signals` | `okapicli.SIGINT`, `okapicli.SIGTERM` |
| `OnStart` | called just before the server starts |
| `OnStarted` | called shortly after a successful start |
| `OnShutdown` | called before graceful shutdown begins |

`RunServer` starts the server in a goroutine, blocks on the signal channel, and shuts down gracefully — no manual `signal.Notify` needed.

### Configuration File Loading

```go
var cfg AppConfig
if err := cli.LoadConfig("config.yaml", &cfg); err != nil { panic(err) }
// supports .json / .yaml / .yml
```

### Flag Retrieval

```go
cli.GetString(name) string
cli.GetInt(name) int
cli.GetBool(name) bool
cli.GetFloat(name) float64
cli.GetDuration(name) time.Duration
cli.Get(name) any
cli.MustParse() *CLI             // Parse or panic
cli.Okapi() *okapi.Okapi         // Underlying Okapi instance
cli.MatchedCommand() *Command    // Matched subcommand after Execute()
```

### Combining CLI With a Config File

Common pattern — CLI overrides go on top of a YAML config:

```bash
./myapp --config=config.prod.yaml --port=9000 --debug
```
