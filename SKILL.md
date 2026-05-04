---
name: Gin
slug: gin-developer
version: 1.0.0
description: Production-grade Gin (Go) web framework development — build robust APIs and web apps without common pitfalls.
metadata:
  {"emoji":"🐹","requires":{"bins":["go"]},"os":["linux","darwin","win32"]}
---

## Navigation

| Topic | File | When to Use |
|---|---|---|
| Router, routing groups, middleware | `routing.md` | Any endpoint setup |
| HTTP methods, JSON binding, validation | `handlers.md` | Building endpoints |
| JSON, XML, YAML, streaming responses | `responses.md` | API response patterns |
| JWT, session, bcrypt, auth middleware | `auth.md` | User auth & permissions |
| MySQL, PostgreSQL, Redis, GORM, migrations | `database.md` + `migrations.md` | Data access |
| Structured logging, middleware, recovery | `middleware.md` | Cross-cutting concerns |
| Goroutines, sync.WaitGroup, context | `concurrency.md` | Parallel processing |
| Context propagation, cancellation, values | `context.md` | Context usage |
| File uploads, multipart, S3, presigned URLs | `file-uploads.md` | Media handling |
| Deployment, Docker, systemd, signals | `deployment.md` | Going live |
| Unit tests, httptest, mocking | `testing.md` | Test-driven dev |
| Security headers, CORS, rate limiting | `security.md` | Hardening |
| Go module versioning, best practices | `versions.md` | Version management |

## Critical Rules (Never Forget)

- **`context.Context` is mandatory** — pass it through every function that does I/O
- **Never spawn goroutines without `sync.WaitGroup` or context cancellation**
- **Check errors always** — `if err != nil { return err }` is not optional
- **`gin.H{}` is just `map[string]interface{}`** — no magic, no hidden behavior
- **Route order matters** — first match wins, put specific routes BEFORE generic ones
- **Binding tags use `validate:""`** not `binding:""` for Gin v1.9+
- **`c.Next()` in middleware** — call it to execute downstream handlers
- **Abort early** with `c.AbortWithStatusJSON()` when auth fails
- **Use `c.Copy()` if you need to access context after async work**
- **Environment variables** — use `os.Getenv()` or `viper`/`envconfig`, never hardcode

## Go Version Defaults

- **Go 1.26** (current stable — 2026)
- **Gin v1.12** (latest — HTTP/3 support, Protocol Buffers, BSON render)
- **Go modules** — always use `go mod init` and `go.mod`
- **nil pointer checks** — always check before dereferencing

## Pro Tips

- `gin.Default()` includes Logger and Recovery middleware — use it in dev
- `gin.New()` is minimal — add only the middleware you need in production
- Use `c.Params` to get path parameters — don't parse URLs manually
- `c.GetHeader("X-Request-ID")` to read request headers
- `gin.Mode()` to set mode: `gin.DebugMode`, `gin.ReleaseMode`, `gin.TestMode`
- Use `httptest.NewRecorder` for testing handlers without a real server

Start with `routing.md` for most common Gin work.