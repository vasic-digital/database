# CLAUDE.md - Database Module

## INHERITED FROM constitution/CLAUDE.md

All rules in `constitution/CLAUDE.md` (and the `constitution/Constitution.md` it references) apply unconditionally. This file's rules below extend them — they MUST NOT weaken any inherited rule. See parent root `CLAUDE.md` §6.AD for the Lava-specific incorporation context (29th §6.L cycle, 2026-05-14) and §6.AD-debt for the implementation-gap inventory. Use `constitution/find_constitution.sh` from the parent project root to resolve the absolute path of the submodule from any nested location.

## INHERITED FROM the Helix Constitution

This module is governed by the Helix Constitution. All rules in the
constitution's `CLAUDE.md` and the `Constitution.md` it references apply
unconditionally. Locate the constitution from any nested depth via its
`find_constitution.sh` helper — do NOT hardcode a path (this module stays
fully decoupled and project-agnostic per §11.4.28).

Canonical reference: https://github.com/HelixDevelopment/HelixConstitution


## Definition of Done

This module inherits the parent project's universal Definition of Done — see the root
`CLAUDE.md` and `docs/development/definition-of-done.md`. In one line: **no
task is done without pasted output from a real run of the real system in the
same session as the change.** Coverage and green suites are not evidence.

### Acceptance demo for this module

```bash
# Real DB: connect → migrate → CRUD → transaction via SQLite pure-Go driver
cd Database && GOMAXPROCS=2 nice -n 19 go test -count=1 -race -v ./tests/integration/...
```
Expect: PASS; exercises `sqlite.New`, `migration.NewRunner`, `repository.Repository[T]`, and the query builder per `Database/README.md` Quick Start. For PostgreSQL set `POSTGRES_URL` and re-run with the `-tags=pgx` build tag.


## Overview

`digital.vasic.database` is a generic, reusable Go module for relational database operations. It provides driver-agnostic interfaces with PostgreSQL and SQLite adapters, connection pooling, schema migrations, a generic repository pattern, and a fluent query builder.

**Module**: `digital.vasic.database` (Go 1.24+)

## Build & Test

```bash
go build ./...
go test ./... -count=1 -race
go test ./... -short              # Unit tests only
go test -tags=integration ./...   # Integration tests (requires PostgreSQL)
go test -bench=. ./...            # Benchmarks
```

## Code Style

- Standard Go conventions, `gofmt` formatting
- Imports grouped: stdlib, third-party, internal (blank line separated)
- Line length <= 100 chars
- Naming: `camelCase` private, `PascalCase` exported, acronyms all-caps
- Errors: always check, wrap with `fmt.Errorf("...: %w", err)`
- Tests: table-driven, `testify`, naming `Test<Struct>_<Method>_<Scenario>`

## Package Structure

| Package | Purpose |
|---------|---------|
| `pkg/database` | Core interfaces (Database, Tx, Row, Rows, Result, Config) |
| `pkg/postgres` | PostgreSQL adapter using pgx/v5 with connection pooling |
| `pkg/sqlite` | SQLite adapter using modernc.org/sqlite (pure Go, no CGO) |
| `pkg/pool` | Generic connection pool with metrics and health checking |
| `pkg/migration` | Schema migration runner with version tracking and rollback |
| `pkg/repository` | Generic repository pattern with CRUD and listing |
| `pkg/query` | Fluent SQL query builder with type-safe conditions |
| `pkg/connection` | Dialect-aware `*sql.DB` wrapper that transparently rewrites queries (placeholders, `INSERT OR IGNORE`, boolean literals) for cross-database compatibility |
| `pkg/dialect` | Cross-database SQL compatibility helpers (SQLite ↔ PostgreSQL): placeholder syntax, DDL differences, auto-increment, timestamp types |
| `pkg/gorm` | GORM adapter wrapping `*gorm.DB` with the pool config, health-check, and transaction helpers shared by the rest of the module |
| `pkg/helpers` | Transaction utilities — primarily a safe-transaction wrapper that auto-commits or rolls back based on the supplied function's outcome |
| `pkg/netstorage` | Entity types and interfaces mirroring the Database-KMP Kotlin module for shared network-storage definitions |

## Key Interfaces

- `database.Database` -- Connect, Close, Exec, Query, QueryRow, Begin, HealthCheck
- `database.Tx` -- Commit, Rollback, Exec, Query, QueryRow
- `pool.Pool` -- Acquire, Release, Stats, Close
- `repository.Repository[T]` -- Create, GetByID, Update, Delete, List, Count
- `repository.EntityMapper[T]` -- TableName, Columns, ScanRow, InsertSQL, UpdateSQL
- `query.Condition` -- Build() (sql, args)

## Dependencies

- `github.com/jackc/pgx/v5` -- PostgreSQL driver
- `modernc.org/sqlite` -- Pure Go SQLite driver
- `github.com/stretchr/testify` -- Testing assertions

## Integration Seams

| Direction | Sibling modules |
|-----------|-----------------|
| Upstream (this module imports) | none |
| Downstream (these import this module) | HelixLLM |

*Siblings* means other project-owned modules at the parent project repo root. The root parent-project app and external systems are not listed here — the list above is intentionally scoped to module-to-module seams, because drift *between* sibling modules is where the "tests pass, product broken" class of bug most often lives. See root `CLAUDE.md` for the rules that keep these seams contract-tested.
