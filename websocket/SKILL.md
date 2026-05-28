## Okapi WebSocket

WebSocket support ships as a separate, framework-agnostic package: `github.com/jkaninda/okapi-ws` (import alias commonly `okapiws`). It works with Okapi handlers and plain `net/http`.

### Installation

```shell
go get github.com/jkaninda/okapi-ws
```

### Helper: Upgrade Inside an Okapi Handler

```go
package main

import (
    "log"
    "net/http"

    "github.com/jkaninda/okapi"
    okapiws "github.com/jkaninda/okapi-ws"
)

func WebSocket(config *okapiws.WSConfig, c *okapi.Context) (*okapiws.WSConnection, error) {
    upgrader := okapiws.NewWSUpgrader(config)
    return upgrader.Upgrade(c.Response(), c.Request(), nil)
}

func WebSocketWithHeaders(config *okapiws.WSConfig, headers http.Header, c *okapi.Context) (*okapiws.WSConnection, error) {
    upgrader := okapiws.NewWSUpgrader(config)
    return upgrader.Upgrade(c.Response(), c.Request(), headers)
}
```

### Echo Handler

```go
func main() {
    app := okapi.Default()

    app.Get("/", func(c *okapi.Context) error {
        return c.OK(okapi.M{"message": "Hello from Okapi!"})
    })

    app.Get("/ws", handleWebSocket)

    if err := app.Start(); err != nil { panic(err) }
}

func handleWebSocket(c *okapi.Context) error {
    ws, err := WebSocket(nil, c) // nil = default config
    if err != nil { return err }
    defer func() {
        if err := ws.Close(); err != nil {
            log.Printf("error closing WebSocket: %v", err)
        }
    }()

    ws.OnMessage(func(msg *okapiws.WSMessage) {
        log.Printf("[%d] %s", msg.Type, msg.Data)
        _ = ws.Send(msg.Data) // echo back
    })

    ws.OnError(func(err error) {
        log.Printf("WebSocket error: %v", err)
    })

    ws.Start()

    <-ws.Context().Done() // block until closed
    return nil
}
```

### With Plain `net/http`

```go
func main() {
    http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
        upgrader := okapiws.NewWSUpgrader(nil)
        ws, err := upgrader.Upgrade(w, r, nil)
        if err != nil {
            http.Error(w, "WebSocket upgrade failed", http.StatusBadRequest)
            return
        }
        defer ws.Close()

        ws.OnMessage(func(msg *okapiws.WSMessage) {
            log.Printf("[%d] %s", msg.Type, msg.Data)
            _ = ws.Send(msg.Data)
        })

        ws.OnError(func(err error) { log.Printf("WebSocket error: %v", err) })
        ws.Start()

        <-ws.Context().Done()
    })

    log.Println("Listening on :8080…")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Detecting a WebSocket Upgrade

```go
if c.IsWebSocketUpgrade() {
    // Connection: Upgrade + Upgrade: websocket headers present
}
```

`okapi.LoggerMiddleware` automatically skips logging for WebSocket upgrade requests so the access log isn't polluted with the long-lived connection.

### Connection API (Conventions)

- `ws.OnMessage(func(*WSMessage))` — message callback
- `ws.OnError(func(error))` — error callback
- `ws.OnClose(func())` — close callback
- `ws.Send(data []byte)` / `ws.SendText(text string)` / `ws.SendJSON(v any)` — write outbound
- `ws.Start()` — start the read loop
- `ws.Close()` — close the connection
- `ws.Context()` — `context.Context` cancelled on close

### Tips

- Always defer `ws.Close()` after a successful upgrade.
- Block on `<-ws.Context().Done()` to keep the handler alive for the lifetime of the connection.
- Hold conversation state in a wrapping struct, not on the `*Context` (which is per-request, not per-connection).
- Use the standard ping/pong pattern for connection liveness; configure intervals via the `WSConfig` passed to `NewWSUpgrader`.
