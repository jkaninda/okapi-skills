## Okapi WebSocket (`okapiws`)

WebSocket support ships as a separate, framework-agnostic package. It works with Okapi handlers and with plain `net/http`.

### Installation

The repository is `jkaninda/okapi-ws`, but the **module path is `github.com/jkaninda/okapiws`** — use that in `go get` and in imports:

```shell
go get github.com/jkaninda/okapiws
```

```go
import okapiws "github.com/jkaninda/okapiws"
```

Built on `github.com/gorilla/websocket`.

### Upgrading Inside an Okapi Handler

`c.Response()` implements `Hijack`, so the upgrader works directly:

```go
func WebSocket(config *okapiws.WSConfig, c *okapi.Context) (*okapiws.WSConnection, error) {
    upgrader := okapiws.NewWSUpgrader(config) // nil config = defaults
    return upgrader.Upgrade(c.Response(), c.Request(), nil)
}

func WebSocketWithHeaders(config *okapiws.WSConfig, headers http.Header, c *okapi.Context) (*okapiws.WSConnection, error) {
    upgrader := okapiws.NewWSUpgrader(config)
    return upgrader.Upgrade(c.Response(), c.Request(), headers)
}
```

Or upgrade with defaults in one call: `okapiws.Default(c.Response(), c.Request(), nil)`.

### Echo Server

```go
func main() {
    app := okapi.Default()

    app.Get("/", func(c *okapi.Context) error {
        return c.OK(okapi.M{"message": "Hello from Okapi!"})
    })

    app.Get("/ws", handleWebSocket)

    if err := app.Start(); err != nil {
        panic(err)
    }
}

func handleWebSocket(c *okapi.Context) error {
    ws, err := WebSocket(nil, c)
    if err != nil {
        return err
    }
    defer func() {
        if err := ws.Close(); err != nil {
            log.Printf("error closing WebSocket: %v", err)
        }
    }()

    ws.OnMessage(func(msg *okapiws.WSMessage) {
        log.Printf("[%d] %s", msg.Type, msg.Data)
        _ = ws.Send(msg.Data) // echo back
    })

    ws.OnError(func(err error) { log.Printf("WebSocket error: %v", err) })
    ws.OnClose(func() { log.Println("client disconnected") })

    ws.Start()            // start the read/write pumps

    <-ws.Context().Done() // block until the connection closes
    return nil
}
```

### With Plain `net/http`

```go
http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
    upgrader := okapiws.NewWSUpgrader(nil)
    ws, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        http.Error(w, "WebSocket upgrade failed", http.StatusBadRequest)
        return
    }
    defer ws.Close()

    ws.OnMessage(func(msg *okapiws.WSMessage) { _ = ws.Send(msg.Data) })
    ws.OnError(func(err error) { log.Printf("WebSocket error: %v", err) })
    ws.Start()

    <-ws.Context().Done()
})

log.Fatal(http.ListenAndServe(":8080", nil))
```

### Server API

```go
okapiws.NewWSUpgrader(config *WSConfig) *WSUpgrader
upgrader.Upgrade(w http.ResponseWriter, r *http.Request, responseHeader http.Header) (*WSConnection, error)
okapiws.Default(w, r, responseHeader) (*WSConnection, error)  // upgrade with default settings
okapiws.DefaultWSConfig() *WSConfig
```

`*WSConnection`:

```go
ws.OnMessage(func(*okapiws.WSMessage))  // message callback
ws.OnError(func(error))                 // error callback
ws.OnClose(func())                      // close callback
ws.Send(data []byte) error              // text frame (non-blocking)
ws.SendText(text string) error
ws.SendBinary(data []byte) error
ws.SendJSON(v any) error
ws.SendEvent(event string, data any) error // {event, data} envelope
ws.Start()                              // start read/write goroutines
ws.Close() error                        // graceful close
ws.IsClosed() bool
ws.Context() context.Context            // cancelled when the connection closes
```

`WSMessage`: `Type int`, `Data []byte`, `Event string`, `Error error`.

### WSConfig

| Field | Default |
|-------|---------|
| `ReadBufferSize` / `WriteBufferSize` | 1024 |
| `HandshakeTimeout` | 10s |
| `CheckOrigin func(*http.Request) bool` | `func(*http.Request) bool { return true }` — allows **any** origin; override in production |
| `Subprotocols` | none |
| `EnableCompression` | false |
| `PingInterval` | 54s |
| `PongWait` | 60s |
| `WriteWait` | 10s |
| `MaxMessageSize` | 512 KB |

```go
cfg := &okapiws.WSConfig{
    CheckOrigin:    func(r *http.Request) bool { return r.Header.Get("Origin") == "https://app.example.com" },
    PingInterval:   30 * time.Second,
    PongWait:       40 * time.Second,
    MaxMessageSize: 1 << 20,
}
ws, err := okapiws.NewWSUpgrader(cfg).Upgrade(c.Response(), c.Request(), nil)
```

Ping/pong keep-alive is handled for you from `PingInterval` / `PongWait`.

### Client

The package also ships a client with optional auto-reconnect:

```go
client := okapiws.NewWSClient("wss://api.example.com/ws",
    okapiws.WithConfig(&okapiws.WSClientConfig{
        AutoReconnect:    true,
        ReconnectInitial: time.Second,
        ReconnectMax:     30 * time.Second,
        MaxRetries:       0, // unlimited
        Headers:          http.Header{"Authorization": {"Bearer " + token}},
    }))

client.OnConnect(func() { log.Println("connected") })       // fires on connect and each reconnect
client.OnMessage(func(msg *okapiws.WSMessage) { log.Printf("%s", msg.Data) })
client.OnError(func(err error) { log.Println(err) })
client.OnClose(func() { log.Println("closed") })

if err := client.Connect(ctx); err != nil {
    return err
}
defer client.Close()

_ = client.SendJSON(map[string]any{"type": "subscribe", "topic": "prices"})
<-client.Context().Done()
```

`WSClientConfig` adds `TLSConfig`, `Subprotocols`, `EnableCompression`, and the reconnect knobs to the shared buffer/timeout fields. `okapiws.DefaultWSClient()` returns the defaults.

### Detecting an Upgrade Request

```go
if c.IsWebSocketUpgrade() {
    // Connection: Upgrade + Upgrade: websocket present
}
```

`okapi.LoggerMiddleware` skips WebSocket upgrades, so a long-lived connection does not sit in the access log.

### Tips

- Always `defer ws.Close()` after a successful upgrade.
- Block on `<-ws.Context().Done()` to keep the handler alive for the connection's lifetime.
- Keep per-connection state in your own struct — `*okapi.Context` is per-request, not per-connection.
- Set `CheckOrigin` explicitly — the default accepts every origin, which is fine for development only.
