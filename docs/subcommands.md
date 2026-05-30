# Subcommands

The CLI matches the standard WXYC cache-builder shape:

- **`wikidata-cache build INPUT [--data-dir DIR] [--limit N] [--progress-interval N] [--gzip] [--resume] [--state-file FILE]`** — streams the JSON dump and writes the 8 CSV files. `--resume`/`--state-file` come from `wxyc_etl::cli::ResumableBuildArgs`; the streaming filter is idempotent so they are accepted but currently no-ops.
- **`wikidata-cache import [--data-dir DIR] [--database-url URL] [--fresh]`** — loads the 8 CSV files into PostgreSQL. The connection URL falls back to the `DATABASE_URL_WIKIDATA` env var via `wxyc_etl::cli::resolve_database_url`.

`--output-dir` (build) and `--csv-dir` (import) are accepted for one release as deprecated aliases for `--data-dir` and emit a stderr warning.

The `import` subcommand:

1. Creates the schema (idempotent with `IF NOT EXISTS`)
2. Sets tables to UNLOGGED for faster bulk import
3. Truncates existing data
4. Streams each CSV via COPY TEXT in FK order (entity first, then child tables)
5. Restores tables to LOGGED
6. Runs VACUUM FULL

The `--fresh` flag drops and recreates the schema before importing.
