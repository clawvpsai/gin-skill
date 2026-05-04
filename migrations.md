# Database Migrations — Go/Gin Schema Management

## Plain SQL Migrations (Recommended for Go)

Go projects typically use raw SQL migration files rather than ORM-generated schemas.

**Directory structure:**
```
migrations/
  001_create_users.sql
  002_create_posts.sql
  003_add_users_email_index.sql
  down/
  001_drop_users.sql
  002_drop_posts.sql
```

**Migration file:**
```sql
-- 001_create_users.up.sql
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);
```

**Down migration:**
```sql
-- 001_create_users.down.sql
DROP TABLE IF EXISTS users;
```

## Running Migrations (golang-migrate)

```bash
go get -u github.com/golang-migrate/migrate/v4/cmd/migrate
```

```bash
# Create migration
migrate create -ext sql -dir migrations -seq create_users

# Run up
migrate -path migrations -database "postgres://localhost:5432/myapp?sslmode=disable" up

# Run down
migrate -path migrations -database "postgres://localhost:5432/myapp?sslmode=disable" down 1

# Force version (if migration state is inconsistent)
migrate -path migrations -database "postgres://localhost:5432/myapp?sslmode=disable" force 001
```

**In Go code:**
```go
import (
    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func RunMigrations(db *sql.DB) error {
    m, err := migrate.New(
        "file://migrations",
        "postgres://postgres:password@localhost:5432/myapp?sslmode=disable",
    )
    if err != nil {
        return fmt.Errorf("failed to create migrate instance: %w", err)
    }
    defer m.Close()

    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return fmt.Errorf("failed to run migrations: %w", err)
    }

    return nil
}
```

## GORM Auto-Migrate (Development Only)

```go
import "gorm.io/gorm"

func autoMigrate(db *gorm.DB) error {
    return db.AutoMigrate(
        &User{},
        &Post{},
        &Comment{},
    )
}
```

**⚠️ NEVER use AutoMigrate in production** — it drops and recreates columns, losing data.

## Common Column Types (PostgreSQL)

| Type | Go Type | Notes |
|---|---|---|
| `BIGSERIAL` | `uint` | Auto-increment bigint |
| `SERIAL` | `uint` | Auto-increment int |
| `VARCHAR(n)` | `string` | Variable length |
| `TEXT` | `string` | Unlimited text |
| `BOOLEAN` | `bool` | True/false |
| `TIMESTAMP` | `time.Time` | Date + time |
| `TIMESTAMPTZ` | `time.Time` | With timezone |
| `INTEGER` | `int` | 32-bit int |
| `BIGINT` | `int64` | 64-bit int |
| `DECIMAL(p,s)` | `float64` | Exact precision |
| `JSONB` | `json.RawMessage` | Binary JSON |
| `UUID` | `uuid.UUID` | Unique identifier |
| `BYTEA` | `[]byte` | Binary data |

## MySQL-Specific

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**MySQL migrate:**
```bash
migrate -path migrations -database "mysql://user:password@tcp(localhost:3306)/myapp?charset=utf8mb4&parseTime=True" up
```

## Indexes

```sql
-- Single column
CREATE INDEX idx_users_email ON users(email);

-- Composite
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);

-- Unique
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Partial (PostgreSQL)
CREATE INDEX idx_posts_published ON posts(user_id) WHERE published_at IS NOT NULL;

-- Drop
DROP INDEX IF EXISTS idx_users_email;
```

## Common Mistakes

1. **AutoMigrate in production** — data loss risk; use versioned SQL migrations
2. **No down migrations** — always be able to rollback
3. **Long migration steps** — break into multiple small migrations
4. **Not using IF NOT EXISTS** — migration fails on repeat run
5. **Missing indexes on FK columns** — slow joins; GORM doesn't auto-add FK indexes
6. **Wrong charset** — always use `utf8mb4` for MySQL, handles emoji correctly

## Updated from Research (2026-05)
- golang-migrate is the standard SQL migration tool for Go projects
- GORM v1.31+ supports `migrate.WithForeignKeyRemoved` for cleaner schema management

Source: [golang-migrate](https://github.com/golang-migrate/migrate) | [GORM Migrations](https://gorm.io/docs/migration.html)
