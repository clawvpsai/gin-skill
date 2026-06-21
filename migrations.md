# Database Migrations — Go/Gin Schema Management

## Choosing a Migration Tool (Decision Framework)

Go projects have four common migration strategies. Pick one and stick with it — mixing tools creates version-table conflicts.

| Tool | Style | Best For | Notable Feature |
|---|---|---|---|
| **golang-migrate** | Versioned SQL files | Teams that want a mature, widely-adopted CLI with the broadest DB support | Single binary, embed-friendly, ~30 DB drivers |
| **goose** (v3.27.1, Apr 2026) | Versioned SQL + Go functions | Teams that want to run complex migrations in Go code (transactions, branching logic) | `Provider` API with custom stores, Aurora DSQL support |
| **Atlas** (v1.2.0, Apr 2026) | Declarative (schema-as-code) **or** versioned | Teams that want Terraform-like workflows with drift detection and CI linting | Lints destructive changes, supports `golang-migrate`/`goose`/`dbmate`/`flyway`/`liquibase` directory formats |
| **GORM AutoMigrate** | Auto-sync from structs | Prototypes only — **never** production | Diffs Go struct → schema |

**Recommendation for Gin services in 2026:**
- New service, simple SQL → **golang-migrate**
- Need to mix SQL and Go logic in migrations → **goose**
- Multiple services, CI/CD with policy enforcement → **Atlas**
- Prototyping → GORM AutoMigrate, then migrate to versioned before prod

Source: [Bytebase 2026 Migration Tools Roundup](https://www.bytebase.com/blog/top-database-schema-change-tool-evolution/), [Codelit comparison](https://codelit.io/blog/database-migration-tools-comparison)

---

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

## Running Migrations (golang-migrate) — v4.19.1 (Nov 2025)

```bash
go install github.com/golang-migrate/migrate/v4/cmd/migrate@v4.19.1
```

```bash
# Create migration (sequential numbering)
migrate create -ext sql -dir migrations -seq create_users

# Create migration (timestamp prefix — recommended for parallel dev branches)
migrate create -ext sql -dir migrations -format "20060102150405" create_users

# Run up
migrate -path migrations -database "postgres://localhost:5432/myapp?sslmode=disable" up

# Run down
migrate -path migrations -database "postgres://localhost:5432/myapp?sslmode=disable" down 1

# Force version (if migration state is inconsistent)
migrate -path migrations -database "postgres://localhost:5432/myapp?sslmode=disable" force 001

# Check current version (no changes applied)
migrate -path migrations -database "postgres://localhost:5432/myapp?sslmode=disable" version
```

**In Go code — embedded migrations for single-binary deployment:**

Use `go:embed` so migrations ship inside your binary. No need to copy `.sql` files into Docker images or Kubernetes pods.

```go
import (
    "embed"
    "github.com/golang-migrate/migrate/v4"
    "github.com/golang-migrate/migrate/v4/database/postgres"
    "github.com/golang-migrate/migrate/v4/source/iofs"
)

//go:embed migrations/*.sql
var migrationFS embed.FS

func RunMigrations(db *sql.DB) error {
    driver, err := postgres.WithInstance(db, &postgres.Config{})
    if err != nil {
        return fmt.Errorf("postgres driver: %w", err)
    }

    // iofs source reads from an embed.FS — works inside a binary
    src, err := iofs.New(migrationFS, "migrations")
    if err != nil {
        return fmt.Errorf("iofs source: %w", err)
    }

    m, err := migrate.NewWithInstance("iofs", src, "postgres", driver)
    if err != nil {
        return fmt.Errorf("migrate instance: %w", err)
    }
    // No m.Close() needed — WithInstance does not own the DB connection

    if err := m.Up(); err != nil && !errors.Is(err, migrate.ErrNoChange) {
        return fmt.Errorf("run migrations: %w", err)
    }

    return nil
}
```

**Why `iofs` over `file://`?** `file://` requires migrations on disk at runtime. `iofs` lets you ship a single static binary with the migrations baked in — perfect for distroless container images and minimal Docker layers.

## Goose — v3.27.1 (Apr 24, 2026)

goose is the strongest alternative when you need Go function migrations (data backfills, branching logic) alongside SQL.

**Key changes in v3.27.x:**
- **Minimum Go version is now 1.25** — won't build on Go 1.24 or earlier
- SQL templates no longer auto-include `StatementBegin`/`StatementEnd` annotations (only needed for stored procedures)
- Added Aurora DSQL dialect
- Exposed dialect `Querier` interface
- Added `WithLogger` to `NewProvider` for custom loggers (e.g., `slog`)

**Install:**
```bash
go install github.com/pressly/goose/v3/cmd/goose@latest
```

**Migration file — up + down in one file with annotations:**
```sql
-- +goose Up
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- +goose StatementBegin
CREATE OR REPLACE FUNCTION trigger_set_updated_at() RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- +goose StatementEnd

-- +goose Down
DROP TABLE IF EXISTS users;
DROP FUNCTION IF EXISTS trigger_set_updated_at();
```

**Go-based migrations (for backfills, complex logic):**
```go
package migrations

import (
    "database/sql"
    "github.com/pressly/goose/v3"
)

// 00002_backfill_user_slugs.go
func init() {
    goose.AddMigrationContext(upBackfillSlugs, downBackfillSlugs)
}

func upBackfillSlugs(ctx context.Context, tx *sql.Tx) error {
    // Batch update — never load everything into memory
    batchSize := 1000
    var lastID int64
    for {
        result, err := tx.ExecContext(ctx, `
            UPDATE users SET slug = LOWER(REPLACE(email, '@', '-'))
            WHERE id > $1 AND slug IS NULL
            ORDER BY id ASC
            LIMIT $2
        `, lastID, batchSize)
        if err != nil {
            return err
        }
        rows, _ := result.RowsAffected()
        if rows == 0 {
            break
        }
        // Get max ID in this batch for next iteration
        if err := tx.QueryRowContext(ctx,
            `SELECT COALESCE(MAX(id), 0) FROM users WHERE slug IS NOT NULL AND id > $1`,
            lastID).Scan(&lastID); err != nil {
            return err
        }
    }
    return nil
}

func downBackfillSlugs(ctx context.Context, tx *sql.Tx) error {
    _, err := tx.ExecContext(ctx, `UPDATE users SET slug = NULL`)
    return err
}
```

**Goose Provider API (v3.16+, recommended for new code):**
```go
import (
    "database/sql"
    "embed"
    "github.com/pressly/goose/v3"
)

//go:embed migrations/*.sql
var embedMigrations embed.FS

func RunMigrations(db *sql.DB) error {
    goose.SetBaseFS(embedMigrations)
    
    // WithLogger hooks into stdlib slog
    slogLogger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    goose.SetLogger(slogLogger)

    if err := goose.SetDialect("postgres"); err != nil {
        return err
    }
    return goose.Up(db, "migrations")
}
```

Source: [pressly/goose releases](https://github.com/pressly/goose/releases), [goose v3.27.1 docs](https://pkg.go.dev/github.com/pressly/goose/v3)

## Atlas — Declarative Schema-as-Code (v1.2.0, Apr 10 2026)

Atlas is "Terraform for databases." It works in two modes:

### Mode 1: Declarative (state-based)

Define your desired schema, Atlas computes the diff and applies it.

**`schema.hcl`:**
```hcl
schema "main" {}

table "users" {
  schema = schema.main
  column "id" {
    type = bigserial
    null = false
  }
  column "email" {
    type = varchar(255)
    null = false
    unique = true
  }
  column "created_at" {
    type = timestamptz
    null = false
    default = sql("CURRENT_TIMESTAMP")
  }
  primary_key {
    columns = [column.id]
  }
  index "idx_users_email" {
    unique = true
    columns = [column.email]
  }
}
```

**`atlas.hcl`:**
```hcl
env "dev" {
  url = "postgres://postgres:pass@localhost:5432/myapp?sslmode=disable"
  dev_url = "docker://postgres/16/dev"
  migration {
    dir = "file://migrations"
  }
}
```

**Workflow:**
```bash
# Inspect current DB → HCL
atlas schema inspect -u "postgres://..." -o schema.hcl

# Diff live DB vs. desired state, generate migration plan
atlas schema diff \
  --from "postgres://..." \
  --to file://schema.hcl \
  --dev-url "docker://postgres/16/dev"

# Apply with approval
atlas schema apply \
  --url "postgres://..." \
  --to file://schema.hcl \
  --dev-url "docker://postgres/16/dev"

# Lint for destructive changes in CI
atlas migrate lint \
  --dir file://migrations \
  --dev-url "docker://postgres/16/dev"
```

### Mode 2: Versioned (auto-planned)

Atlas diffs the current schema vs. your desired state and **writes** the SQL migration file for you. Works with GORM models via [`atlas-provider-gorm`](https://github.com/ariga/atlas-provider-gorm).

```bash
# Generate a migration by diffing live DB against GORM models
atlas migrate diff add_user_logs \
  --dir "file://migrations" \
  --to "file://schema.hcl" \
  --dev-url "docker://postgres/16/dev"
```

### Use Your Existing golang-migrate/goose Files with Atlas

Atlas reads other tools' migration directory formats out of the box — no conversion needed:

```hcl
env "dev" {
  url = "postgres://..."
  dev_url = "docker://postgres/16/dev"
  migration {
    dir = "file://migrations?format=golang-migrate"  # or: goose, dbmate, flyway, liquibase
  }
}
```

The `format` flag tells Atlas there are `.up.sql`/`.down.sql` pairs (or `-- +goose Up`/`-- +goose Down` annotations) so it doesn't try to run destructive rollbacks on its own.

**Why Atlas for Gin services:**
- **CI lint** — `atlas migrate lint` fails the build if a migration drops a column with data, takes an `ACCESS EXCLUSIVE` lock >5s, or adds a column without a default
- **Drift detection** — alerts when the live DB diverges from version-controlled schema
- **Go SDK** — `pkg.go.dev/ariga.io/atlas-go-sdk/atlasexec` for programmatic migrations from your Gin startup hook
- **GitHub Actions / GitLab CI / Kubernetes Operator** — first-class integrations

Source: [atlasgo.io](https://atlasgo.io/), [Atlas + golang-migrate format](https://atlasgo.io/faq/migrations-with-golang)

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

**⚠️ NEVER use AutoMigrate in production** — it drops and recreates columns, losing data. Even in dev, it can't add column defaults, change types, or run data backfills.

**AutoMigrate gotchas:**
- Does **not** add column defaults — you must set them via raw SQL after
- Drops indexes that don't match exactly (case-sensitive collation matters)
- Renames look like drop+create → all data lost

## Common Column Types (PostgreSQL)

| Type | Go Type | Notes |
|---|---|---|
| `BIGSERIAL` | `uint` | Auto-increment bigint |
| `SERIAL` | `uint` | Auto-increment int |
| `VARCHAR(n)` | `string` | Variable length |
| `TEXT` | `string` | Unlimited text |
| `BOOLEAN` | `bool` | True/false |
| `TIMESTAMP` | `time.Time` | Date + time |
| `TIMESTAMPTZ` | `time.Time` | With timezone — **preferred** over `TIMESTAMP` |
| `INTEGER` | `int` | 32-bit int |
| `BIGINT` | `int64` | 64-bit int |
| `DECIMAL(p,s)` | `float64` | Exact precision (use `shopspring/decimal` for money) |
| `JSONB` | `json.RawMessage` | Binary JSON — faster queries than `JSON` |
| `UUID` | `uuid.UUID` | Unique identifier (Go 1.27+ stdlib `uuid` package) |
| `BYTEA` | `[]byte` | Binary data |

**Go 1.27+ UUID generation (stdlib):**
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),  -- requires pgcrypto or PG 13+
    ...
);
```

```go
import "uuid"  // Go 1.27+ stdlib — replaces github.com/google/uuid
id := uuid.NewRandomV7()  // UUIDv7 — time-ordered, better DB index locality
```

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

-- Composite (column order matters — leftmost prefix rule)
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);

-- Unique
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Partial (PostgreSQL) — smaller index, faster writes
CREATE INDEX idx_posts_published ON posts(user_id) WHERE published_at IS NOT NULL;

-- Expression index (PostgreSQL) — index on computed value
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Drop
DROP INDEX IF EXISTS idx_users_email;
```

## Zero-Downtime Migrations (Expand-Contract Pattern)

A breaking schema change (`ALTER TABLE users RENAME COLUMN name TO full_name`) takes an `ACCESS EXCLUSIVE` lock — your Gin service stops accepting writes until it completes. For zero-downtime, split every breaking change into three deploys:

**Phase 1 — Expand** (add new shape, keep old working):
```sql
-- Migration 1: Expand
ALTER TABLE users ADD COLUMN full_name text;
ALTER TABLE users ADD CONSTRAINT users_full_name_required CHECK (full_name IS NOT NULL) NOT VALID;
-- Add NOT VALID so the constraint isn't enforced on existing rows
```

**Phase 2 — Migrate** (backfill in batches, deploy app code that dual-writes):
```sql
-- Migration 2: Backfill in batches (run in a goroutine, not at startup)
DO $$
DECLARE
    last_id bigint := 0;
    affected int;
BEGIN
    LOOP
        UPDATE users
        SET full_name = name
        WHERE id > last_id AND full_name IS NULL
        ORDER BY id ASC
        LIMIT 5000;
        GET DIAGNOSTICS affected = ROW_COUNT;
        EXIT WHEN affected = 0;
        SELECT MAX(id) INTO last_id FROM users WHERE full_name IS NOT NULL AND id > last_id;
    END LOOP;
END $$;
```
Then deploy app code that writes `full_name` on every INSERT/UPDATE.

**Phase 3 — Switch reads** (gradual cutover via feature flag):
```go
// Use full_name for 1% of reads → 10% → 50% → 100%
if rand.Float64() < fullNameReadPercent {
    user.Name = user.FullName
}
```

**Phase 4 — Contract** (after verification, drop old column):
```sql
-- Migration 3: Contract (safe to roll forward only — never roll back)
ALTER TABLE users DROP COLUMN name;
```

**Tools that automate this:**
- **[squawk](https://github.com/squaredup/squawk)** — PostgreSQL migration linter, fails CI on dangerous patterns (e.g., adding `NOT NULL` without default)
- **[pgroll](https://github.com/xataio/pgroll)** — Xata's automated expand-contract for PostgreSQL
- **[reshape](https://github.com/fairwindsops/reshape)** — auto-generates expand-contract migrations
- **[pg_repack](https://reorg.github.io/pg_repack/)** — rebuilds tables without `ACCESS EXCLUSIVE` locks
- **Atlas `migrate lint`** — same as squawk, baked into Atlas workflows

**Forbidden patterns (CI should reject):**
```sql
-- ❌ Locks table, rewrites every row
ALTER TABLE users ALTER COLUMN name TYPE varchar(500);

-- ✅ Add new column, backfill, switch, drop old
ALTER TABLE users ADD COLUMN name_v2 varchar(500);
-- ... backfill ...
ALTER TABLE users DROP COLUMN name;

-- ❌ Adds NOT NULL without default — fails on existing rows
ALTER TABLE users ADD COLUMN status text NOT NULL;

-- ✅ Nullable, or with safe default
ALTER TABLE users ADD COLUMN status text NOT NULL DEFAULT 'active';
```

Source: [Zero-Downtime PostgreSQL Migrations 2026](https://dev.to/young_gao/zero-downtime-database-migrations-patterns-for-production-postgresql-in-2026-2a3i), [Expand-Contract pattern (Tim Wellhausen)](https://www.tim-wellhausen.de/papers/ExpandAndContract/ExpandAndContract.html)

## Migration in Gin Startup — Best Practice

Run migrations **before** binding the HTTP port, not in a goroutine. Migrations should be fast and deterministic; if they take >10s, that's a sign you need expand-contract, not async startup.

```go
func main() {
    // 1. Load config + secrets
    cfg := loadConfig()
    
    // 2. Run migrations (fail fast — don't serve traffic on a broken schema)
    db, err := sql.Open("postgres", cfg.DatabaseURL)
    if err != nil {
        log.Fatal("db open:", err)
    }
    defer db.Close()
    
    if err := RunMigrations(db); err != nil {
        log.Fatal("migrations failed:", err)
    }
    
    // 3. NOW bind HTTP port
    r := gin.Default()
    registerRoutes(r, db)
    
    if err := r.Run(cfg.HTTPAddr); err != nil {
        log.Fatal("http server:", err)
    }
}
```

**Multi-replica deployments** — only one replica should run migrations. Options:
- **Kubernetes Job** — migrations as a separate pod that runs before the Deployment rolls out (init container or pre-upgrade hook)
- **Advisory lock** — `SELECT pg_advisory_lock(987654321)` at migration start, released after; only one replica wins
- **Atlas Operator** — Kubernetes-native, watches CRDs and applies migrations

## Common Mistakes

1. **AutoMigrate in production** — data loss risk; use versioned SQL migrations
2. **No down migrations** — always be able to rollback (or use expand-contract for true zero-downtime)
3. **Long migration steps** — break into multiple small migrations; one schema change per file
4. **Not using IF NOT EXISTS** — migration fails on repeat run
5. **Missing indexes on FK columns** — slow joins; GORM doesn't auto-add FK indexes
6. **Wrong charset** — always use `utf8mb4` for MySQL, handles emoji correctly
7. **Migrations in goroutines at startup** — race conditions; run synchronously before binding the port
8. **No `IF NOT EXISTS` on constraints** — partial re-runs break; idempotency matters for retries
9. **`ALTER TABLE ... ALTER COLUMN TYPE` in production** — rewrites the table; use expand-contract
10. **Skipping `IF VALID` on constraints** — `ADD CONSTRAINT ... NOT VALID` lets you add FKs without scanning existing rows

## Migration Linting in CI (squawk)

Squawk catches dangerous patterns before they hit production. Add to your CI pipeline:

```bash
# Install
npm install -g squawk-cli   # or: brew install squawk

# Lint all migrations
squawk migrations/*.sql

# Example output — fails CI build:
#   migrations/042_add_status.sql:1:1: adding-not-nullable-field
#     New `NOT NULL` column "status" has no default and is not nullable
#     ⚠ This will fail on existing rows
```

**Configure allowed warnings per-file** for intentional cases:
```sql
-- squawk-ignore: adding-not-nullable-field
ALTER TABLE users ADD COLUMN status text NOT NULL;
```

Source: [squawk](https://github.com/squaredup/squawk)

## Updated from Research (2026-06)

### Migration Tool Landscape (2026)
- **golang-migrate v4.19.1** (Nov 2025) — still the de facto standard; ~30 DB drivers; `iofs` source for embedded migrations in single-binary deploys
- **goose v3.27.1** (Apr 2026) — minimum Go 1.25; Provider API with `WithLogger` for `slog` integration; Aurora DSQL support
- **Atlas v1.2.0** (Apr 2026) — declarative schema-as-code and versioned auto-planning; reads `golang-migrate`/`goose`/`dbmate`/`flyway`/`liquibase` directories natively; CI lint with `atlas migrate lint`

### Decision Framework
- Simple SQL on a single DB → **golang-migrate**
- Mix SQL + Go function migrations (data backfills, branching logic) → **goose**
- Multi-service org with CI policy enforcement + drift detection → **Atlas**
- Prototype → GORM AutoMigrate, then migrate to versioned before prod

### Zero-Downtime Patterns
- **Expand-contract** is the standard pattern for breaking changes — three deploys minimum (expand → migrate → contract)
- **squawk** lints migrations in CI; rejects `ADD COLUMN NOT NULL` without default, `ALTER TYPE`, and other table-rewrite operations
- **pgroll**, **reshape** automate expand-contract generation
- **Atlas `migrate lint`** is the all-in-one alternative for projects already using Atlas

### Embedded Migrations
- Both `golang-migrate` (via `iofs`) and `goose` (via `SetBaseFS`) support `go:embed` — ship migrations inside the binary, no Docker COPY needed

### Sources
- [golang-migrate](https://github.com/golang-migrate/migrate)
- [pressly/goose](https://github.com/pressly/goose) · [releases](https://github.com/pressly/goose/releases)
- [Atlas](https://atlasgo.io/) · [Atlas + golang-migrate format](https://atlasgo.io/faq/migrations-with-golang)
- [squawk](https://github.com/squaredup/squawk)
- [Zero-Downtime PostgreSQL Migrations 2026](https://dev.to/young_gao/zero-downtime-database-migrations-patterns-for-production-postgresql-in-2026-2a3i)
- [Expand-Contract pattern](https://www.tim-wellhausen.de/papers/ExpandAndContract/ExpandAndContract.html)
- [Bytebase 2026 Migration Tools Roundup](https://www.bytebase.com/blog/top-database-schema-change-tool-evolution/)
- [GORM Atlas integration](https://gorm.io/docs/migration.html)