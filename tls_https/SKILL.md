## Okapi TLS & HTTPS

### Loading a TLS Config

```go
func okapi.LoadTLSConfig(certFile, keyFile, caFile string, clientAuth bool) (*tls.Config, error)
```

| Parameter | Description |
|-----------|-------------|
| `certFile` | Path to the certificate (PEM) |
| `keyFile` | Path to the private key (PEM) |
| `caFile` | Optional CA certificate for verifying client certs — `""` disables |
| `clientAuth` | Require client certificate verification (mTLS) |

### HTTPS-Only Server

```go
tlsConfig, err := okapi.LoadTLSConfig("cert.pem", "key.pem", "", false)
if err != nil {
    panic(err)
}

o := okapi.New(okapi.WithTLS(tlsConfig), okapi.WithAddr(":8443"))

o.Get("/", func(c *okapi.Context) error {
    return c.OK(okapi.M{"status": "secure"})
})

o.Start() // ListenAndServeTLS
```

When the server's `TLSConfig` is set, `Start()` serves HTTPS on the configured address.

### Dual HTTP + HTTPS

`WithTLSServer` runs a second listener with its own address, sharing the same routes and middleware:

```go
tlsConfig, _ := okapi.LoadTLSConfig("cert.pem", "key.pem", "", false)

o := okapi.Default()                        // HTTP on :8080
o.With(okapi.WithTLSServer(":8443", tlsConfig)) // HTTPS on :8443

o.Start() // HTTP served in a goroutine, HTTPS in the foreground
```

`Stop()` / `StopWithContext(ctx)` gracefully shut down both listeners.

### Mutual TLS (client certificates)

```go
tlsConfig, err := okapi.LoadTLSConfig("server.crt", "server.key", "ca.crt", true)
o := okapi.New(okapi.WithTLS(tlsConfig))
```

With `clientAuth: true`, the CA in `caFile` verifies presented client certificates. Inspect the peer in a handler:

```go
o.Get("/whoami", func(c *okapi.Context) error {
    tls := c.Request().TLS
    if tls == nil || len(tls.PeerCertificates) == 0 {
        return c.AbortUnauthorized("client certificate required")
    }
    return c.OK(okapi.M{"cn": tls.PeerCertificates[0].Subject.CommonName})
})
```

### Custom `*tls.Config` (autocert / Let's Encrypt)

The `*Okapi` struct has no exported fields, so supply a pre-built `*http.Server` instead of assigning to one:

```go
certManager := autocert.Manager{
    Prompt:     autocert.AcceptTOS,
    HostPolicy: autocert.HostWhitelist("example.com", "www.example.com"),
    Cache:      autocert.DirCache("certs"),
}

o := okapi.New(okapi.WithServer(&http.Server{
    Addr:      ":443",
    TLSConfig: certManager.TLSConfig(),
}))

o.Get("/", func(c *okapi.Context) error { return c.OK(okapi.M{"ok": true}) })

// TLSConfig is set, so Start() calls ListenAndServeTLS
go http.ListenAndServe(":80", certManager.HTTPHandler(nil)) // ACME http-01 challenge
o.Start()
```

`okapi.WithTLS(certManager.TLSConfig())` works the same way when you don't need to customise the rest of the server.

### Generating a Development Certificate

```bash
openssl genrsa -out server.key 2048
openssl req -new -x509 -sha256 -key server.key -out server.crt -days 365
```

### HTTP → HTTPS Redirect Middleware

`c.Redirect` writes the redirect and returns nothing, so return `nil` after calling it:

```go
func redirectToHTTPS(c *okapi.Context) error {
    if c.Request().TLS == nil {
        c.Redirect(http.StatusMovedPermanently, "https://"+c.Request().Host+c.Request().RequestURI)
        return nil
    }
    return c.Next()
}

o.Use(redirectToHTTPS)
```

### HSTS

```go
func hsts(c *okapi.Context) error {
    c.SetHeader("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
    return c.Next()
}

o.Use(hsts)
```

### Practices

- Terminate TLS 1.2+ only — set `MinVersion` on a custom `*tls.Config` when you build one yourself.
- Serve HSTS only over HTTPS, and only once you are sure every subdomain supports TLS.
- Keep the ACME http-01 listener on `:80` when using autocert, or use a DNS challenge.
