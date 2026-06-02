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

**Postgres image dependency**: migration `0003_wxyc_identity_match_functions.sql` creates the `wxyc_unaccent` text-search dictionary from `wxyc_unaccent.rules`, which Postgres reads from `$SHAREDIR/tsearch_data/`. The destination PG must run [`ghcr.io/wxyc/wxyc-postgres:pg16`](https://github.com/WXYC/wxyc-etl/blob/main/docs/wxyc-postgres-image.md) (built + published by WXYC/wxyc-etl#127); the image bakes the rules file into the base. CI (`.github/workflows/ci.yml`) and `docker-compose.yml` pin this image. The migration wraps the `CREATE TEXT SEARCH DICTIONARY` call in a plpgsql `EXCEPTION WHEN SQLSTATE 'F0000'` block that re-raises with the operator runbook URL when the rules file is missing — so a stock-image deploy fails fast with an actionable message instead of a bare `config_file_error`. The Railway production PG service must be swapped to the same image one-time (tracked in the runbook).
