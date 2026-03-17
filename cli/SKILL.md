## Okapi CLI Integration (`okapicli` package)

### Basic Usage

```go
import "github.com/jkaninda/okapi/okapicli"

cli := okapicli.New(app, "My API").
    String("config", "c", "config.yaml", "Config file path").
    Int("port", "p", 8000, "Server port").
    Bool("debug", "d", false, "Enable debug mode").
    Float("rate", "r", 1.0, "Rate limit").
    Duration("timeout", "t", 30*time.Second, "Request timeout")

cli.Parse()
app.WithPort(cli.GetInt("port"))
cli.Run()
```

### Struct-Based Configuration

```go
type Config struct {
    Port    int           `cli:"port"    short:"p" desc:"Server port"    env:"PORT"    default:"8080"`
    Debug   bool          `cli:"debug"   short:"d" desc:"Debug mode"     env:"DEBUG"`
    Config  string        `cli:"config"  short:"c" desc:"Config file"    env:"CONFIG"`
    Timeout time.Duration `cli:"timeout" short:"t" desc:"Timeout"        env:"TIMEOUT" default:"30s"`
}

cfg := &Config{}
cli := okapicli.New(app, "My API").FromStruct(cfg)
cli.Parse()
// cfg fields are populated with resolved values (CLI > env > default)
```

### Subcommands

```go
cli.Command("serve", "Start the HTTP server", func(cmd *okapicli.Command) error {
    port := cmd.GetInt("port")
    cmd.Okapi().WithPort(port)
    return cmd.CLI().Run()
}).Int("port", "p", 8080, "HTTP server port")

cli.Command("migrate", "Run database migrations", func(cmd *okapicli.Command) error {
    // migration logic
    return nil
}).String("dsn", "", "", "Database connection string")

cli.DefaultCommand("serve")  // Run "serve" when no subcommand is specified
cli.Execute()
```

### Command Methods

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

### Server Lifecycle via CLI

```go
// Simple run with default options (30s shutdown timeout, SIGINT/SIGTERM)
cli.Run()

// Run with custom options
cli.RunServer(&okapicli.RunOptions{
    ShutdownTimeout: 10 * time.Second,
    Signals:         []os.Signal{okapicli.SIGINT, okapicli.SIGTERM},
    OnStart:         func() { log.Println("Starting...") },
    OnStarted:       func() { log.Println("Server ready") },
    OnShutdown:      func() { log.Println("Shutting down...") },
})
```

### Configuration File Loading

```go
var cfg AppConfig
cli.LoadConfig("config.yaml", &cfg)  // Supports .json, .yaml, .yml
```

### Flag Retrieval

```go
cli.GetString(name) string
cli.GetInt(name) int
cli.GetBool(name) bool
cli.GetFloat(name) float64
cli.GetDuration(name) time.Duration
cli.Get(name) interface{}
cli.MustParse() *CLI             // Parse or panic
cli.Okapi() *okapi.Okapi         // Access underlying Okapi instance
cli.MatchedCommand() *Command    // Get matched subcommand after Execute()
```
