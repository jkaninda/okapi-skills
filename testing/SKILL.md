## Okapi Testing

Two pieces: `okapi`'s test server / test context, and the `okapitest` package's fluent request builder and assertions.

### Test Server

```go
func TestBooks(t *testing.T) {
    server := okapi.NewTestServer(t)          // random free port, stopped via t.Cleanup
    server.Get("/books", GetBooksHandler)     // *TestServer embeds *Okapi — register as usual

    okapitest.GET(t, server.BaseURL+"/books").
        ExpectStatusOK().
        ExpectBodyContains("The Go Programming Language")
}
```

Constructors:

```go
okapi.NewTestServer(t TestingT) *TestServer                    // new Okapi instance
okapi.NewTestServerOn(t TestingT, port int) *TestServer         // fixed port
okapi.NewTestServerWithOkapi(t TestingT, o *Okapi) *TestServer   // wrap a configured instance
okapi.DefaultTestServer(t TestingT) *TestServer                  // okapi.Default() based
```

`*TestServer` embeds `*Okapi` and adds `BaseURL string`.

`TestingT` is satisfied by `*testing.T` (`Helper`, `Cleanup`, `Errorf`, `Fatalf`), so a custom harness can be plugged in.

Starting an already-built app for a test:

```go
o := buildApp()                    // your production wiring
baseURL := o.StartForTest(t)       // starts and registers cleanup
addr := o.WaitForServer(2 * time.Second) // block until ready (when starting manually)
```

### Test Context (unit-testing a handler directly)

```go
ctx, rec := okapi.NewTestContext("POST", "/books", strings.NewReader(`{"name":"Go"}`))
ctx.Request().Header.Set("Content-Type", "application/json")

if err := CreateBookHandler(ctx); err != nil {
    t.Fatal(err)
}

okapitest.FromRecorder(t, rec).
    ExpectStatusCreated().
    ExpectJSONPath("name", "Go")
```

`NewTestContext` builds its own in-memory request and `httptest.ResponseRecorder` without a full Okapi engine.

### Fluent Requests (`okapitest`)

```go
import "github.com/jkaninda/okapi/okapitest"

okapitest.GET(t, url).
    Header("Authorization", "Bearer "+token).
    ExpectStatusOK().
    ExpectContentType("application/json").
    ExpectBodyContains("Go Programming")

okapitest.POST(t, url).
    JSONBody(map[string]any{"name": "Book"}).
    ExpectStatusCreated()
```

Verb entry points: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`, plus `Request(t)` for a blank builder and `FromRecorder(t, rec)` for a recorded response.

### Reusable Client

```go
client := okapitest.NewClient(t, server.BaseURL)
client.Headers["Authorization"] = "Bearer " + token   // default headers for every request

client.GET("/books").ExpectStatusOK()
client.POST("/books").JSONBody(book).ExpectStatusCreated()
```

### Request Builder

```go
rb.Method(method)                      // HTTP method
rb.URL(url)                            // full URL
rb.Path(path)                          // append a path segment
rb.Header(key, value)
rb.Headers(map[string]string{...})
rb.QueryParam(key, value)
rb.QueryParams(map[string]string{...})
rb.SetBasicAuth(user, pass)
rb.SetBearerAuth(token)
rb.Body(io.Reader)                     // raw body
rb.JSONBody(v)                         // marshalled, Content-Type: application/json
rb.FormBody(map[string]string)         // application/x-www-form-urlencoded
rb.Timeout(d)

rb.Execute() (*http.Response, []byte)  // run and inspect manually
```

Assertions are chainable and fail the test through `*testing.T`; the request is issued once on the first assertion.

### Assertions

```go
// Status
rb.ExpectStatus(code)
rb.ExpectStatusOK()                    // 200
rb.ExpectStatusCreated()               // 201
rb.ExpectStatusAccepted()              // 202
rb.ExpectStatusNoContent()             // 204
rb.ExpectStatusBadRequest()            // 400
rb.ExpectStatusUnauthorized()          // 401
rb.ExpectStatusForbidden()             // 403
rb.ExpectStatusNotFound()              // 404
rb.ExpectStatusConflict()              // 409
rb.ExpectStatusInternalServerError()   // 500

// Body
rb.ExpectBody(expected)                // exact match
rb.ExpectBodyContains(substr)
rb.ExpectContains(substr)              // alias of ExpectBodyContains
rb.ExpectBodyNotContains(substr)
rb.ExpectEmptyBody()

// JSON
rb.ExpectJSON(expected)                // deep-equal comparison
rb.ExpectJSONPath("user.profile.name", "Ada") // dot path
rb.ParseJSON(&target)                  // unmarshal into a struct for further checks

// Headers
rb.ExpectHeader(key, value)
rb.ExpectHeaderContains(key, substr)
rb.ExpectHeaderExists(key)
rb.ExpectContentType(contentType)

// Cookies
rb.ExpectCookieExist(name)
rb.ExpectCookie(name, value)
```

### Utilities

```go
okapitest.GracefulExitAfter(d)  // send SIGTERM after d — for shutdown/integration tests

// Deprecated one-shot helpers — prefer the builder:
okapitest.AssertHTTPStatus(t, method, url, headers, body, contentType, expected)
okapitest.AssertHTTPResponse(t, method, url, headers, body, contentType, expectedStatus, expectedBody)
```

### End-to-End Example

```go
func TestCreateAndFetchBook(t *testing.T) {
    server := okapi.NewTestServer(t)
    RegisterRoutes(server.Okapi) // your wiring

    client := okapitest.NewClient(t, server.BaseURL)

    client.POST("/api/books").
        JSONBody(okapi.M{"name": "The Go Programming Language", "price": 30}).
        ExpectStatusCreated().
        ExpectJSONPath("name", "The Go Programming Language")

    client.GET("/api/books").
        ExpectStatusOK().
        ExpectContentType("application/json").
        ExpectBodyContains("The Go Programming Language")

    client.GET("/api/books/999").
        ExpectStatusNotFound()
}
```
