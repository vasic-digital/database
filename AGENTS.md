# AGENTS.md - Database Module Multi-Agent Coordination

## INHERITED FROM constitution/AGENTS.md

All rules in `constitution/AGENTS.md` (and the `constitution/Constitution.md` it references) apply unconditionally. This file's rules below extend them — they MUST NOT weaken any inherited rule. See parent root `CLAUDE.md` §6.AD for the Lava-specific incorporation context (29th §6.L cycle, 2026-05-14) and §6.AD-debt for the implementation-gap inventory. Use `constitution/find_constitution.sh` from the parent project root to resolve the absolute path of the submodule from any nested location.

## INHERITED FROM the Helix Constitution

This module is governed by the Helix Constitution. All rules in the
constitution's `AGENTS.md` and the `Constitution.md` it references apply
unconditionally. Locate the constitution from any nested depth via its
`find_constitution.sh` helper — do NOT hardcode a path (this module stays
fully decoupled and project-agnostic per §11.4.28).

Canonical reference: https://github.com/HelixDevelopment/HelixConstitution

## Module Identity

- **Module**: `digital.vasic.database`
- **Purpose**: Generic, reusable Go module for relational database operations
- **Language**: Go 1.24+
- **Packages**: database, postgres, sqlite, pool, migration, repository, query

## Agent Responsibilities

### Database Core Agent

**Owns**: `pkg/database/`

- Maintains the driver-agnostic interface contracts (`Database`, `Tx`, `Row`, `Rows`, `Result`)
- Maintains `Config` struct and its `DSN()` / `Validate()` methods
- Any change to core interfaces requires coordination with all adapter agents (PostgreSQL, SQLite)
- Must not import any driver-specific packages

### PostgreSQL Agent

**Owns**: `pkg/postgres/`

- Maintains `Client` (implements `database.Database`) backed by `pgx/v5` and `pgxpool`
- Manages `Config` (embeds `database.Config`), `DefaultConfig()`, pool configuration
- Wraps `pgconn.CommandTag`, `pgx.Row`, `pgx.Rows`, `pgx.Tx` into database interfaces
- Exposes `Pool()` for advanced pgxpool access and `Migrate()` for raw SQL migrations
- Integration tests require a live PostgreSQL instance

### SQLite Agent

**Owns**: `pkg/sqlite/`

- Maintains `Client` (implements `database.Database`) backed by `modernc.org/sqlite`
- Manages `Config`, `DefaultConfig(path)`, PRAGMA application (WAL, foreign_keys, synchronous, busy_timeout)
- Wraps `sql.Result`, `*sql.Row`, `*sql.Rows`, `*sql.Tx` into database interfaces
- Exposes `DB()` for advanced `*sql.DB` access
- Pure Go -- no CGO dependency

### Pool Agent

**Owns**: `pkg/pool/`

- Maintains `GenericPool` with semaphore-based concurrency control
- Manages `PoolConfig`, `PoolStats`, `DefaultPoolConfig()`
- Implements connection lifecycle: acquire (idle reuse or factory), release, eviction
- Background health check loop with `ConnHealthChecker` callback
- Tracks metrics atomically: acquire count, errors, latency, peak concurrency
- Functional dependencies: `ConnFactory`, `ConnHealthChecker`, `ConnCloser`

### Migration Agent

**Owns**: `pkg/migration/`

- Maintains `Runner` with version-tracked schema migration management
- Manages `Migration` struct (Version, Name, Up, Down SQL)
- Operations: `Init` (create tracking table), `Applied` (list versions), `Apply` (forward), `Rollback` / `RollbackWith` (reverse)
- Each migration runs in a transaction (apply + record / rollback + delete)
- Depends on `database.Database` interface only

### Repository Agent

**Owns**: `pkg/repository/`

- Maintains `GenericRepository[T]` implementing `Repository[T]` interface
- Manages `EntityMapper[T]` contract for per-entity table/column/scan/SQL mapping
- CRUD operations: `Create`, `GetByID`, `Update`, `Delete`, `List`, `Count`
- `ListOptions` with `WhereClause`, pagination (`Offset`, `Limit`), ordering
- Depends on `database.Database` interface only

### Query Agent

**Owns**: `pkg/query/`

- Maintains `Builder` with fluent method chaining (Select, From, Where, OrderBy, Limit, Offset, GroupBy, Having)
- Maintains `Condition` interface and built-in implementations:
  - Comparison: `Eq`, `Neq`, `Gt`, `Gte`, `Lt`, `Lte`, `Like`
  - Null: `IsNull`, `IsNotNull`
  - Set: `In`
  - Composite: `And`, `Or`
- `Build()` produces parameterized SQL with `?` placeholders
- Zero external dependencies -- standalone SQL generation

## Coordination Rules

### Interface Changes

Any modification to interfaces in `pkg/database/` (Database, Tx, Row, Rows, Result) requires:
1. Database Core Agent updates the interface definition
2. PostgreSQL Agent updates `Client` and wrapper types to satisfy the new contract
3. SQLite Agent updates `Client` and wrapper types to satisfy the new contract
4. Repository Agent and Migration Agent review for downstream impact

### New Adapter

Adding a new database adapter (e.g., MySQL):
1. Database Core Agent verifies no interface changes are needed
2. New adapter agent creates `pkg/<driver>/` implementing `database.Database`
3. Tests must cover Connect, Close, Exec, Query, QueryRow, Begin, HealthCheck

### Repository Mapper

Adding a new entity type:
1. Repository Agent defines the `EntityMapper[T]` implementation
2. No changes required to `GenericRepository` -- it is generic over `T`

### Migration Schema

Adding new migrations:
1. Migration Agent manages the `Migration` structs
2. The `Runner` auto-initializes the tracking table (`schema_migrations`)
3. Each migration version must be unique and monotonically increasing

## Testing Boundaries

| Agent | Unit Tests | Integration Tests | Requirements |
|-------|-----------|-------------------|--------------|
| Database Core | Interface mocks, Config validation | None | None |
| PostgreSQL | Config defaults, DSN building | Full CRUD, transactions | Live PostgreSQL |
| SQLite | Config defaults, PRAGMA verification | Full CRUD, transactions, in-memory | None (pure Go) |
| Pool | Config validation, stats tracking | Acquire/release cycles, eviction | None |
| Migration | Version sorting, applied tracking | Apply/rollback sequences | database.Database impl |
| Repository | WhereClause building | Full CRUD with ListOptions | database.Database impl |
| Query | SQL generation, condition building | None | None |

## Communication Protocol

Agents coordinate through:
- **Interface contracts**: `pkg/database/` defines the shared API surface
- **Go module versioning**: Breaking changes require a major version bump
- **Test coverage**: Every exported function must have table-driven tests with `testify`
- **Error wrapping**: All errors use `fmt.Errorf("context: %w", err)` for traceability
