# Testing

```bash
cargo test              # all tests (unit + CLI + oracle + PG skipped without DB)
cargo test --lib        # unit tests only
cargo test --test cli_tests    # CLI integration tests only
cargo test --test import_test  # PostgreSQL import tests (requires docker compose up -d)

# PostgreSQL integration tests (requires TEST_DATABASE_URL)
TEST_DATABASE_URL=postgresql://musicbrainz:musicbrainz@localhost:5434/postgres \
  cargo test --test pg_import_test
```

- **Unit tests** (26): JSON parsing, filter logic, extractor, CSV output, pipeline output trait.
- **CLI tests** (9): End-to-end binary invocation with small_dump.json fixture, including env-var fallback for `DATABASE_URL_WIKIDATA` and deprecation warnings for renamed flags.
- **Oracle tests** (9): CSV output diffed against expected baselines in `tests/fixtures/expected/`.
- **PG import tests** (13): Full filter -> CSV -> PG import -> query chain. Trigram search on entity names and aliases. Discogs/MusicBrainz ID lookup via indexes. Gated on `TEST_DATABASE_URL`.
- **Import integration tests**: Require PostgreSQL on port 5435 (started via `docker compose up -d`). Cover schema creation, CSV import for all 8 tables, FK integrity, Unicode handling, and end-to-end pipeline validation.

## Build

```bash
cargo build --release   # produces target/release/wikidata-cache
cargo install --path .  # installs to ~/.cargo/bin/
```

## Code Style

- `cargo fmt` for formatting
- `cargo clippy` for linting
- Targets macOS ARM64 and Linux x86_64
