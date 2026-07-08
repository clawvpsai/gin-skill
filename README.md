# 🐹 Gin (Go) Developer Skill

> Production-grade Gin web framework skill for OpenClaw agents — build robust APIs and web apps without common pitfalls.

[![Go 1.26](https://img.shields.io/badge/Go-1.26-blue?style=flat-square)](https://go.dev)
[![Gin v1.12](https://img.shields.io/badge/Gin-v1.12-blue?style=flat-square)](https://github.com/gin-gonic/gin)
[![MIT](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)
[![Auto-updated](https://img.shields.io/badge/Auto--updated-6h-blue?style=flat-square)](#auto-updater)

---

## Why This Skill Exists

Go/Gin is fast-moving — HTTP/3, QUIC, Protocol Buffers, new concurrency patterns. Most AI assistants still suggest `golang-jwt/jwt/v4` (deprecated!) or `c.QueryInt()` (doesn't exist in Gin). This skill is the ground truth — auto-updated and version-current.

---

## 🚀 Quick Start

```bash
# Check Go version
go version

# Check Gin version
go list -m github.com/gin-gonic/gin

# Load SKILL.md first for navigation
# Then dive into the topic you need
```

---

## 📚 Topics Covered (14 files)

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
| File uploads, multipart, S3, presigned | `file-uploads.md` | Media handling |
| Deployment, Docker, systemd, signals | `deployment.md` | Going live |
| Unit tests, httptest, mocking | `testing.md` | Test-driven dev |
| Security headers, CORS, rate limiting | `security.md` | Hardening |
| Go module versioning, best practices | `versions.md` | Version management |

---

## ⚠️ Critical Rules (Never Forget)

| Rule | Why |
|---|---|
| `context.Context` is mandatory | Pass through every I/O function — DB, HTTP, file ops |
| Never spawn goroutines without `sync.WaitGroup` or context cancellation | Memory leaks and race conditions |
| Check errors always | `if err != nil { return err }` — not optional |
| `gin.H{}` is just `map[string]interface{}` | No magic, no hidden behavior |
| Route order matters | First match wins — specific routes BEFORE generic |
| Binding tags use `binding:""` | Gin uses go-playground/validator internally but with `SetTagName("binding")` — NOT `validate:""` |
| Use `c.Copy()` for context after async work | Original context may be cancelled |
| Never use `jwt/v4` | Deprecated — always use `jwt/v5` |

---

## 📦 Version Support

| Version | Status | Key Features |
|---|---|---|
| **Go 1.26** | ✅ Latest (2026) | Improved toolchain, slices/maps packages |
| Go 1.26.5 / 1.25.12 | ✅ Patch (2026-07-07) | CVE-2026-39822 + CVE-2026-42505 fixes — **pin `go.mod` toolchain** |
| Go 1.27rc2 | ⚠️ RC2 (2026-07-07) | Tagged before CVE-2026-56853 fix — **avoid h2c production** |
| Go 1.25 | Previous stable | Toolchain refinements |
| Go 1.24 | Minimum for Gin v1.12 | Generic type aliases, math/rand/v2 |
| **Gin v1.12** | ✅ Latest (Feb 2026) | BSON, ProtoBuf, HTTP/3 via QUIC, UseEscapedPath option

---

## 🔄 Auto-Updater

The skill is **auto-updated every 6 hours** via a cron job. The agent decides whether to update based on whether it finds meaningful gaps — no forced updates.

**How it works:**
1. Checks current state of all 14 topic files
2. Identifies weak or outdated areas via web research
3. Checks for new Go/Gin/dependency releases
4. Updates files with new patterns + source URLs
5. Commits and pushes to `main` branch automatically

---

## 🛠️ Contributing

```bash
# Clone
git clone git@github.com:clawvpsai/gin-skill.git
cd gin-skill

# Edit any .md file — updater picks up changes
# For structural changes, submit a PR
```

All skill files are `.md` — no code generation needed. Just update patterns, add examples, or fix deprecations.

---

## 📊 Skill Stats

- **14 topic files** covering full Gin/Go development lifecycle
- **Version-aware** — Go 1.26, 1.25, 1.24 + Gin v1.12 covered
- **~6,500+ lines** of production-ready content
- **Auto-updated** via OpenClaw cron — never stale
- **MIT licensed** — free for everyone

---

<p align="center">
  <a href="https://github.com/clawvpsai/gin-skill">📂 GitHub Repo</a>
  · <a href="https://gin-gonic.com/docs">📖 Gin Docs</a>
  · <a href="https://go.dev/dl">📋 Go Downloads</a>
</p>
