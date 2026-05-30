# Migrations

Schema changes ship as numbered SQL files under `migrations/`, applied with [sqlx-cli](https://crates.io/crates/sqlx-cli). The baseline `migrations/0001_initial.sql` mirrors `schema/create_database.sql`.

Install sqlx-cli once:

```bash
cargo install sqlx-cli --no-default-features --features postgres
```

Add a new migration:

```bash
sqlx migrate add <descriptive_name>
# edits a new migrations/000N_<descriptive_name>.sql; write forward-only SQL
```

Apply against a database (e.g., a fresh local Postgres):

```bash
sqlx migrate run --database-url postgresql://localhost:5435/<db> --source migrations
```

**Deploy wiring**: `sqlx migrate run` is invoked by `.github/workflows/rebuild-cache.yml` before every monthly rebuild. Every migration in `migrations/` must be idempotent (`CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`, `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`) — re-applying against a populated prod DB is required to be a no-op so the rebuild cron stays safe.

**Dual-write convention**: when adding a schema change, write the new `migrations/000N_*.sql` AND mirror it into `schema/create_database.sql` so fresh-rebuild parity holds. The two paths must produce the same end-state.

The `src/import_schema.rs::apply_schema()` runtime path remains the source of truth for fresh-rebuild DDL; `sqlx migrate run` is the source of truth for incremental schema evolution between rebuilds.
