# Deployment — Docker, Systemd, Signals, Production

## Project Structure

```
myapp/
├── cmd/
│   └── server/
│       └── main.go          # Entry point
├── internal/
│   ├── handlers/
│   ├── models/
│   └── services/
├── pkg/
├── migrations/
├── go.mod
├── go.sum
├── Dockerfile
├── docker-compose.yml
└── .env
```

## Dockerfile (Multi-stage, Go 1.26)

```dockerfile
# Build stage
FROM golang:1.26-alpine AS builder

WORKDIR /app

# Install dependencies
COPY go.mod go.sum ./
RUN go mod download

# Copy source
COPY . .

# Build — disable CGO for static binary
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o /server ./cmd/server

# Runtime stage
FROM alpine:3.21

WORKDIR /app

# Install CA certs and timezone data
RUN apk --no-cache add ca-certificates tzdata

# Copy binary from builder
COPY --from=builder /server .
COPY --from=builder /app/migrations ./migrations

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

ENTRYPOINT ["./server"]
```

## Docker Compose (Local Dev)

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://postgres:password@db:5432/myapp?sslmode=disable
      - REDIS_URL=redis://cache:6379
    depends_on:
      - db
      - cache
    volumes:
      - .:/app
    command: ["air"] # hot reload with air

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data

  cache:
    image: redis:7-alpine
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

## Docker Compose (Production)

```yaml
version: '3.8'

services:
  app:
    build: .
    restart: always
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - JWT_SECRET=${JWT_SECRET}
      - ENV=production
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    volumes:
      - /var/log/myapp:/app/logs
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M

  db:
    image: postgres:16-alpine
    restart: always
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    restart: always
    command: redis-server --appendonly yes
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

## Systemd Service

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Gin Application
After=network.target postgres.target redis.target
Wants=postgres.service redis.service

[Service]
Type=simple
User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/server
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

# Environment
EnvironmentFile=/opt/myapp/.env

# Graceful shutdown
KillSignal=SIGTERM
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp
sudo systemctl status myapp

# View logs
journalctl -u myapp -f
```

## Graceful Shutdown

```go
package main

import (
    "context"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
    
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    // ... routes setup ...
    
    srv := &http.Server{
        Addr:    ":8080",
        Handler: r,
    }
    
    // Start server in goroutine
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen: %s\n", err)
        }
    }()
    
    log.Printf("Server started on :8080")
    
    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    
    log.Println("Shutting down server...")
    
    // Give outstanding requests 30 seconds to complete
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("Server forced to shutdown: %v", err)
    }
    
    log.Println("Server exited")
}
```

## Nginx Reverse Proxy

```nginx
server {
    listen 80;
    server_name api.myapp.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.myapp.com;

    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;

    client_max_body_size 10M;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }
}
```

## Health Check Endpoint

```go
func healthHandler(c *gin.Context) {
    // Check DB
    sqlDB, err := DB.DB()
    if err != nil || sqlDB.Ping() != nil {
        c.JSON(503, gin.H{"status": "unhealthy", "db": "down"})
        return
    }
    
    // Check Redis
    if _, err := redisClient.Ping(c).Result(); err != nil {
        c.JSON(503, gin.H{"status": "unhealthy", "redis": "down"})
        return
    }
    
    c.JSON(200, gin.H{
        "status": "ok",
        "version": "1.0.0",
        "time": time.Now().UTC(),
    })
}

r.GET("/health", healthHandler)
r.GET("/healthz", healthHandler) // Kubernetes style
```

## Environment Variables

```bash
# .env (never commit this file)
DATABASE_URL=postgres://user:pass@localhost:5432/myapp?sslmode=disable
REDIS_URL=redis://localhost:6379
JWT_SECRET=your-super-secret-key-at-least-32-chars
ENV=development

# Production — use real values from secrets manager
ENV=production
```

## Common Mistakes

1. **No graceful shutdown** — requests die abruptly on restart
2. **Running as root in Docker** — create and use non-root user
3. **No health check endpoint** — load balancers can't detect unhealthy instances
4. **No resource limits in Docker** — container can consume all memory
5. **Not setting `GIN_MODE=release`** — debug mode leaks stack traces
6. **No log rotation** — disk fills up from logs
7. **No CPU/memory limits** — a single bad query can starve the entire service

---

## Updated from Research (2026-05)

### Docker Improvements
- **Alpine 3.21** is now preferred over 3.19 (security fixes, updated CA certs)
- **`-ldflags="-w -s"`** — strips debug info, reduces binary size by ~15%
- **Resource limits** in docker-compose `deploy:` section — prevents one service from consuming all host resources

### Multi-stage Build Optimizations
- `go mod download` before `COPY . .` — better layer caching (deps change less often than source)
- `CGO_ENABLED=0` — produces fully static binary, no libc dependency

### Sources
- Go 1.26 container image: `golang:1.26-alpine`
- Alpine 3.21 release notes