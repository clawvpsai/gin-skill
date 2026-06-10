# Database — MySQL, PostgreSQL, Redis, GORM

## GORM Setup

```go
package db

import (
    "fmt"
    "os"
    "time"
    
    "gorm.io/driver/mysql"
    "gorm.io/driver/postgres"
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

var DB *gorm.DB

func InitDB() error {
    dsn := os.Getenv("DATABASE_URL")
    
    var err error
    DB, err = gorm.Open(postgres.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info),
    })
    
    if err != nil {
        return fmt.Errorf("failed to connect to database: %w", err)
    }
    
    sqlDB, err := DB.DB()
    if err != nil {
        return err
    }
    
    // Connection pool
    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetMaxOpenConns(100)
    sqlDB.SetConnMaxLifetime(time.Hour)
    
    return nil
}

func Close() error {
    sqlDB, err := DB.DB()
    if err != nil {
        return err
    }
    return sqlDB.Close()
}
```

## GORM Models

```go
type User struct {
    ID        uint      `gorm:"primaryKey" json:"id"`
    Name      string    `gorm:"size:100;not null" json:"name"`
    Email     string    `gorm:"uniqueIndex;size:255;not null" json:"email"`
    Password  string    `gorm:"size:255;not null" json:"-"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"-"` // soft delete
    Posts     []Post    `gorm:"foreignKey:AuthorID" json:"posts,omitempty"`
}

type Post struct {
    ID        uint           `gorm:"primaryKey" json:"id"`
    Title     string         `gorm:"size:200;not null" json:"title"`
    Body      string         `gorm:"type:text" json:"body"`
    AuthorID  uint           `gorm:"not null;index" json:"author_id"`
    Author    User           `gorm:"foreignKey:AuthorID" json:"author,omitempty"`
    Tags      []Tag          `gorm:"many2many:post_tags" json:"tags,omitempty"`
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"-"`
}
```

## GORM CRUD

```go
// Create
user := User{Name: "John", Email: "john@example.com"}
result := DB.Create(&user) // INSERT

// Batch create
users := []User{{Name: "John"}, {Name: "Jane"}}
result := DB.CreateInBatches(users, 100)

// Read
var user User
DB.First(&user, 1)                        // by primary key
DB.First(&user, "email = ?", "john@example.com") // where clause

var users []User
DB.Where("age > ?", 18).Find(&users)
DB.Where("name IN ?", []string{"John", "Jane"}).Find(&users)
DB.Not("name = ?", "John").Find(&users)
DB.Or("age > ?", 30).Find(&users)

// Select specific columns
DB.Select("name", "email").Find(&users)

// Count
var count int64
DB.Model(&User{}).Where("age > ?", 18).Count(&count)

// Update
DB.Model(&user).Update("name", "John Updated")
DB.Model(&user).Updates(User{Name: "New", Email: "new@example.com"})
DB.Model(&user).Where("active = ?", true).Update("email", "new@example.com")

// Delete (soft delete if using DeletedAt)
DB.Delete(&user, 1) // DELETE FROM users WHERE id = 1

// Soft delete
DB.Delete(&user) // UPDATE users SET deleted_at WHERE id = 1

// Hard delete (ignore soft delete)
DB.Unscoped().Delete(&user)
```

## Preloading (Eager Loading)

```go
// Preload relationships (avoids N+1)
var posts []Post
DB.Preload("Author").Find(&posts)

// Nested preload
DB.Preload("Author.Profile").Find(&posts)

// Preload with conditions
DB.Preload("Posts", "active = ?", true).Find(&users)

// Joins (for filtering by relationship)
var posts []Post
DB.Joins("JOIN users ON users.id = posts.author_id").
    Where("users.active = ?", true).
    Find(&posts)
```

## Transactions

```go
// Transaction
err := DB.Transaction(func(tx *gorm.DB) error {
    user := User{Name: "John"}
    if err := tx.Create(&user).Error; err != nil {
        return err
    }
    
    post := Post{Title: "Hello", AuthorID: user.ID}
    if err := tx.Create(&post).Error; err != nil {
        return err
    }
    
    return nil
})
```

## Raw SQL

```go
// Use sparingly — parameterized queries only
var result struct {
    Name  string
    Count int
}
DB.Raw("SELECT name, COUNT(*) as count FROM users GROUP BY name").Scan(&result)

DB.Exec("UPDATE users SET active = ? WHERE id = ?", true, 1)
```

## Redis Client (go-redis v9)

```go
import (
    "github.com/redis/go-redis/v9"
    "context"
)

var redisClient *redis.Client

func InitRedis() error {
    redisClient = redis.NewClient(&redis.Options{
        Addr:     os.Getenv("REDIS_URL"),
        Password: os.Getenv("REDIS_PASSWORD"),
        DB:       0,
    })
    
    _, err := redisClient.Ping(context.Background()).Result()
    return err
}

// Usage
func cacheUser(ctx context.Context, userID int, user *User) error {
    key := fmt.Sprintf("user:%d", userID)
    data, err := json.Marshal(user)
    if err != nil {
        return err
    }
    return redisClient.Set(ctx, key, data, 15*time.Minute).Err()
}

func getCachedUser(ctx context.Context, userID int) (*User, error) {
    key := fmt.Sprintf("user:%d", userID)
    data, err := redisClient.Get(ctx, key).Bytes()
    if err == redis.Nil {
        return nil, nil // cache miss
    }
    if err != nil {
        return nil, err
    }
    
    var user User
    if err := json.Unmarshal(data, &user); err != nil {
        return nil, err
    }
    return &user, nil
}

// Pipeline for batch operations
func batchCacheUsers(ctx context.Context, users []User) error {
    pipe := redisClient.Pipeline()
    for _, user := range users {
        key := fmt.Sprintf("user:%d", user.ID)
        data, _ := json.Marshal(user)
        pipe.Set(ctx, key, data, 15*time.Minute)
    }
    _, err := pipe.Exec(ctx)
    return err
}

// Distributed lock
func acquireLock(ctx context.Context, key string, ttl time.Duration) (bool, error) {
    ok, err := redisClient.SetNX(ctx, "lock:"+key, "1", ttl).Result()
    return ok, err
}

func releaseLock(ctx context.Context, key string) error {
    return redisClient.Del(ctx, "lock:"+key).Err()
}
```

## Common Mistakes

1. **N+1 queries** — always use `Preload()` for relationships
2. **Not setting connection pool** — DB runs out of connections under load
3. **String interpolation in raw SQL** — SQL injection vector, use `?` placeholders
4. **Ignoring `err` from GORM operations** — always check errors
5. **Using `*gorm.DB` not passed through context** — can't cancel long queries
6. **Forgetting to close DB connection** — call `db.Close()` on shutdown
7. **Not using pipelines for batch ops** — multiple round-trips to Redis instead of one

---


## ⚠️ Redis Security — May 2026 Critical CVEs

Redis server 8.x had **5 critical/high severity CVEs** patched in Redis 8.6.3 (May 5, 2026):

| CVE | Severity | Impact |
|---|---|---|
| CVE-2026-23479 | Critical | Use-After-Free in unblock client flow → RCE |
| CVE-2026-25243 | Critical | Invalid memory access in RESTORE → RCE |
| CVE-2026-25588 | Critical | Lua Use-After-Free → RCE |
| CVE-2026-25589 | High | Additional Lua memory issue |
| CVE-2026-23631 | High | Additional RESTORE/RDB issue |

**Action items:**
- **Upgrade Redis server to 8.6.4+ immediately** — 8.6.3 (May CVE patch) superseded by 8.6.4 (June 4, 2026) with additional critical bug fixes
- These are server-side vulnerabilities — the `go-redis/v9` client is not affected
- If using Redis in a Go deployment, ensure your **Redis server** (not the client lib) is patched
- Monitor [Redis security advisories](https://redis.io/docs/latest/operate/rc/security/) for your deployment method (Redis Open Source, Redis Stack, or Redis Enterprise)

**Redis 8.6.4** (June 4, 2026) — bug fix release superseding 8.6.3. Critical fixes include use-after-free, AArch64 startup crash on Linux, Lua debugger under-copy, cluster crash on `CLIENT KILL` inside `EXEC`, `XREADGROUP` consumer replication inconsistency, TCP deadlock fixes, and `SCAN` integer overflow in `COUNT` parameter.

**Redis 8.8.0** (May 25, 2026) — latest minor in the 8.x line with general improvements.

```go
// Verify Redis server version in health checks
func RedisHealthCheck(ctx context.Context) error {
    ver, err := redisClient.Info(ctx, "server").Result()
    if err != nil {
        return err
    }
    // Parse "redis_version:X.X.X" from ver string
    // Reject if version < 8.6.4
    return nil
}
```

## Updated from Research (2026-06-10)

### Redis 8.6.4 (June 4, 2026)
- Supersedes 8.6.3 (May CVE patch) with additional critical bug fixes
- Affects production Redis 8.x deployments — upgrade required
- Key fixes: use-after-free (#15059), AArch64 startup crash, Lua debugger under-copy, cluster crash in `EXEC`, `XREADGROUP` inconsistency, TCP stalls/deadlocks (#14667, #14886)

### go-redis v9 Pipelines & Locks
- **Pipeline** usage now recommended for batch Redis operations — reduces round trips significantly
- **Distributed locking** patterns with `SetNX` for coordination across instances
- Latest stable: **v9.20.0** (2026-05-28)

### GORM v1.31 Updates
- **GORM v1.31.1** (Nov 2025) — ensure compatibility with latest GORM
- Transaction callback pattern is stable and preferred over manual session management

### Sources
- https://github.com/redis/redis/releases
- https://github.com/go-gorm/gorm/releases
- Redis 8.6.4 release notes (GitHub)

