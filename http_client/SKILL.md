## Okapi HTTP Client (`okapi/client` package)

`github.com/jkaninda/okapi/client` is a small fluent HTTP client with:

- A request builder per verb
- Default + per-request middleware chain
- Built-in retry policy with exponential backoff
- Body encoders for JSON, XML, YAML, form, multipart, raw
- Response decoders driven by `Content-Type`
- Zero dependency on the Okapi server package — usable against any REST API

### Quick Start

```go
import "github.com/jkaninda/okapi/client"

c := client.New("https://api.example.com",
    client.WithBearerToken(token),
    client.WithUserAgent("my-app/1.0"),
    client.WithTimeout(10*time.Second),
)

var user User
resp, err := c.Get("/users/42").
    WithContext(ctx).
    QueryParam("expand", "profile").
    Do()
if err != nil { return err }
if err := resp.Error(); err != nil { return err } // *client.HTTPError on non-2xx
if err := resp.JSON(&user); err != nil { return err }
```

`Do()` and `Send()` are aliases. For the common "do, decode, fail on non-2xx" path use `Decode`:

```go
var user User
err := c.Get("/users/42").Decode(&user)
```

`Decode` chooses JSON / XML / YAML based on the response `Content-Type`.

### Client Options

| Option | Purpose |
|--------|---------|
| `WithHTTPClient(*http.Client)` | Provide a pre-configured `http.Client` (TLS, transport) |
| `WithTimeout(d)` | Default per-request timeout |
| `WithHeader(k, v)` | Add one default header |
| `WithHeaders(map)` | Merge multiple default headers |
| `WithBearerToken(token)` | Sets `Authorization: Bearer <token>` |
| `WithBasicAuth(u, p)` | Sets `Authorization: Basic ...` |
| `WithUserAgent(ua)` | Sets the default `User-Agent` |
| `WithMiddleware(mw...)` | Append middleware to the chain |
| `WithRetry(policy)` | Default retry policy |

### Request Builder

Each verb returns a `*RequestBuilder`:

```go
resp, err := c.Post("/items").
    WithContext(ctx).
    Header("X-Trace-Id", traceID).
    QueryParam("dry_run", "true").
    JSONBody(Item{Title: "hello"}).
    Timeout(5*time.Second).
    Do()
```

Available verbs: `Get`, `Post`, `Put`, `Patch`, `Delete`, `Head`, `Options`.

### Terminal Methods

| Method | Behavior |
|--------|----------|
| `Do()` | Issue the request, return `(*Response, error)` |
| `Send()` | Alias for `Do()` |
| `Decode(target)` | `Do()` + decode into `target`; returns `*HTTPError` on non-2xx |

### Body Encoders

| Method | Content-Type |
|--------|-------------|
| `JSONBody(v any)` | `application/json` |
| `XMLBody(v any)` | `application/xml` |
| `YAMLBody(v any)` | `application/yaml` |
| `FormBody(map[string]string)` | `application/x-www-form-urlencoded` |
| `Multipart(func(*multipart.Writer) error)` | `multipart/form-data; boundary=…` |
| `RawBody([]byte)` | unset (use `Header` to set) |
| `Body(io.Reader)` | unset (use `Header` to set) |

### Per-Request Auth Shortcuts

```go
c.Get("/me").BearerToken(jwt).Send()
c.Get("/admin").BasicAuth("user", "pass").Send()
```

### Per-Request Overrides

Builders can override client defaults for a single call:

```go
c.Get("/big").
    Timeout(30 * time.Second).
    Retry(client.RetryPolicy{MaxAttempts: 5, BaseDelay: 100 * time.Millisecond}).
    Middleware(client.LoggingMiddleware(os.Stdout)).
    Do()
```

### Response

```go
resp.IsSuccess()             // 2xx?
resp.Error()                 // *HTTPError on non-2xx, nil otherwise
resp.String()                // body as string
resp.Body                    // []byte
resp.Decode(&target)         // format chosen from Content-Type
resp.JSON(&target)
resp.XML(&target)
resp.YAML(&target)
resp.JSONPath("user.profile.name") // dot-path lookup in a JSON object
resp.Cookie("sid")           // *http.Cookie or nil
resp.Header                  // *http.Header
resp.StatusCode              // int
```

### Middleware

```go
type RoundTripFunc func(*http.Request) (*http.Response, error)
type Middleware    func(next RoundTripFunc) RoundTripFunc
```

Order: client middlewares are outermost; per-request middlewares run next; the retry middleware sits innermost.

Built-in middlewares:

| Middleware | Behavior |
|------------|----------|
| `LoggingMiddleware(io.Writer)` | One line per request (method, URL, status, duration) |
| `UserAgentMiddleware(ua)` | Forces `User-Agent` on every request |
| `RequestIDMiddleware()` | Sets `X-Request-Id` (random hex) if absent |

Custom middleware:

```go
auth := func(next client.RoundTripFunc) client.RoundTripFunc {
    return func(req *http.Request) (*http.Response, error) {
        req.Header.Set("X-Service-Token", currentServiceToken())
        return next(req)
    }
}
c := client.New(baseURL, client.WithMiddleware(auth))
```

### Retry Policy

```go
c := client.New(baseURL, client.WithRetry(client.RetryPolicy{
    MaxAttempts: 4,
    BaseDelay:   100 * time.Millisecond,
    MaxDelay:    2 * time.Second,
}))
```

Defaults:

- `MaxAttempts <= 1` → no retries
- `RetryOnStatus` nil → retries on `408`, `429`, `500`, `502`, `503`, `504`
- `MaxDelay == 0` → backoff doubles indefinitely
- Transport errors (network failures) always retry while attempts remain

Custom retry predicate:

```go
client.RetryPolicy{
    MaxAttempts: 3,
    BaseDelay:   50 * time.Millisecond,
    ShouldRetry: func(resp *http.Response, err error) bool {
        return err != nil || (resp != nil && resp.StatusCode == http.StatusBadGateway)
    },
}
```

Request bodies are buffered once and rewound between attempts, so retries work for POST/PUT/PATCH. Backoff is interrupted when the request context is cancelled.

### Errors

```go
resp, err := c.Get("/missing").Do()
if err != nil {
    return err // transport / build error
}
if err := resp.Error(); err != nil {
    var hErr *client.HTTPError
    if errors.As(err, &hErr) {
        fmt.Println(hErr.StatusCode, string(hErr.Body))
    }
    return err
}
```

`Do` (and its alias `Send`) never returns `HTTPError` — a non-2xx response is a valid response. Opt in via `resp.Error()` or `Decode`, which calls `Error()` internally.
