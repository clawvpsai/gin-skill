# Gin (Go) Developer Skill

Production-grade Gin web framework skill for OpenClaw agents.

## Overview

This skill provides comprehensive guidance for building robust APIs and web applications with Gin (Go). It covers routing, middleware, database integration, authentication, and production deployment.

## Navigation

| Topic | File | When to Use |
|---|---|---|
| Router, routing groups, middleware | `routing.md` | Any endpoint setup |
| HTTP methods, JSON binding, validation | `handlers.md` | Building endpoints |
| JSON, XML, YAML, streaming responses | `responses.md` | API response patterns |
| JWT, session, bcrypt, auth middleware | `auth.md` | User auth & permissions |
| MySQL, PostgreSQL, Redis, GORM | `database.md` | Data access |
| Structured logging, middleware, recovery | `middleware.md` | Cross-cutting concerns |
| Goroutines, sync.WaitGroup, context | `concurrency.md` | Parallel processing |
| Deployment, Docker, systemd, signals | `deployment.md` | Going live |
| Unit tests, httptest, mocking | `testing.md` | Test-driven dev |
| Security headers, CORS, rate limiting | `security.md` | Hardening |

## Critical Rules

- **`context.Context` is mandatory** — pass it through every function that does I/O
- **Never spawn goroutines without `sync.WaitGroup` or context cancellation**
- **Check errors always** — `if err != nil { return err }` is not optional
- **`gin.H{}` is just `map[string]interface{}`** — no magic, no hidden behavior
- **Route order matters** — first match wins, put specific routes BEFORE generic ones
- **Binding tags use `validate:""`** not `binding:""` for validator v10+
- **`c.Next()` in middleware** — call it to execute downstream handlers
- **Abort early** with `c.AbortWithStatusJSON()` when auth fails
- **Use `c.Copy()` if you need to access context after async work**
- **Environment variables** — use `os.Getenv()` or `viper`/`envconfig`, never hardcode

## Version Support

- **Go 1.22** (latest) — range-over-func, improved HTTP routing
- **Go 1.21** — log/slog built-in structured logging
- **Gin v1.10+** (stable)

## Auto-Updater

This skill is auto-updated every 2 hours via a cron job that:
1. Rotates through topic areas (routing, handlers, auth, etc.)
2. Scans for gaps and deprecations in current content
3. Performs web research on latest best practices
4. Commits improvements directly to this repository

## Contributing

Edits to skill files are automatically picked up by the updater. To make structural changes, submit a PR to this repository.

## License

MIT