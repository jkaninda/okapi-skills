## Okapi Testing

### Test Server

```go
server := okapi.NewTestServer(t)
server.Get("/books", handler)
// server.BaseURL gives the test server URL
```

### Test Context (unit tests)

```go
ctx, recorder := okapi.NewTestContext("GET", "/books", nil)
```

### Fluent HTTP Client (`okapitest` package)

```go
import "github.com/jkaninda/okapi/okapitest"

okapitest.GET(t, url).
    Header("Authorization", "Bearer token").
    ExpectStatusOK().
    ExpectBodyContains("Go Programming").
    ExpectHeader("Content-Type", "application/json")

okapitest.POST(t, url).
    JSONBody(map[string]any{"name": "Book"}).
    ExpectStatus(201)
```

### TestClient (reusable with base URL + default headers)

```go
client := okapitest.NewClient(t, server.BaseURL)
client.GET("/books").ExpectStatusOK()
client.POST("/books").JSONBody(book).ExpectStatusCreated()
```

### Request Builder Methods

```go
// Request construction
rb.Method(method)                      // Set HTTP method
rb.URL(url)                            // Set full URL
rb.Path(path)                          // Append path segment
rb.Header(key, value)                  // Set single header
rb.Headers(map[string]string{...})     // Set multiple headers
rb.QueryParam(key, value)              // Add query parameter
rb.QueryParams(map[string]string{...}) // Add multiple query parameters
rb.SetBasicAuth(username, password)    // Set Basic auth header
rb.SetBearerAuth(token)               // Set Bearer auth header
rb.Body(reader)                        // Set raw body
rb.JSONBody(data)                      // Set JSON body (auto-marshal)
rb.FormBody(values)                    // Set form-encoded body
rb.Timeout(duration)                   // Set request timeout

// Execution
rb.Execute() (*http.Response, []byte)  // Execute and return raw response
```

### Response Assertions

```go
// Status codes
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
rb.ExpectBody(expected)                // Exact match
rb.ExpectBodyContains(substr)          // Contains substring
rb.ExpectContains(substr)              // Alias for ExpectBodyContains
rb.ExpectBodyNotContains(substr)       // Does not contain
rb.ExpectEmptyBody()                   // Body is empty

// JSON
rb.ExpectJSON(expected)                // Deep-equal JSON comparison
rb.ExpectJSONPath("path.to.field", v)  // Assert specific JSON path value
rb.ParseJSON(&target)                  // Unmarshal response into struct

// Headers
rb.ExpectHeader(key, value)            // Exact header value match
rb.ExpectHeaderContains(key, substr)   // Header value contains substring
rb.ExpectHeaderExists(key)             // Header is present
rb.ExpectContentType(contentType)      // Shortcut for Content-Type header

// Cookies
rb.ExpectCookieExist(name)             // Cookie exists with non-empty value
rb.ExpectCookie(name, value)           // Cookie has exact value
```

### FromRecorder (for direct handler testing)

```go
okapitest.FromRecorder(t, recorder).
    ExpectStatusOK().
    ExpectBodyContains("success")
```

### Utilities

```go
okapitest.GracefulExitAfter(duration)  // Send SIGTERM after duration (for integration tests)
```
