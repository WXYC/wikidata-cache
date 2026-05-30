# Claude Code Instructions for wikidata-cache

## Project Overview

Purpose-built Rust tool that builds the WXYC `wikidata-cache` PostgreSQL database from Wikidata JSON dumps. Two subcommands matching the standardized WXYC cache-builder CLI shape (`wxyc_etl::cli`): `build` streams a (gzipped) Wikidata JSON dump and writes 8 CSV files of music-relevant entities to `--data-dir`; `import` loads those CSVs into PostgreSQL. Analogous to [discogs-xml-converter](https://github.com/WXYC/discogs-xml-converter) for Discogs data and [musicbrainz-cache](https://github.com/WXYC/musicbrainz-cache) for MusicBrainz.

## Topic guides

CLAUDE.md is a router for the always-loaded reference card. Topic depth lives in `docs/`:

- **[`docs/architecture.md`](docs/architecture.md)** — Module-by-module purpose (`model`, `filter`, `extractor`, `writer`, `import`, `import_schema`, `main`), three-stage parallel pipeline, 8-file CSV output contract, PostgreSQL schema (pg_trgm indexes, FK shape), music-relevance filter criteria (P1953/P1902/P106/P31 primaries vs. secondaries)
- **[`docs/observability.md`](docs/observability.md)** — `wxyc_etl::logger::init` setup, required tag taxonomy (`repo`, `tool`, `step`, `run_id`), optional Sentry forwarding via `SENTRY_DSN`
- **[`docs/subcommands.md`](docs/subcommands.md)** — `build` and `import` CLI shape, `--data-dir` with `--output-dir`/`--csv-dir` deprecated aliases, `DATABASE_URL_WIKIDATA` env fallback, `--fresh` reset, the 6-step `import` flow (UNLOGGED toggle, FK-order COPY, VACUUM FULL)
- **[`docs/scheduling.md`](docs/scheduling.md)** — Monthly rebuild cron (`rebuild-cache.yml`), staggering against discogs-etl/musicbrainz-cache, runner-capacity caveat (130 GB dump vs. GHA's 14 GB disk + 6 h timeout — `workflow_dispatch` is the supported path until self-hosted/Railway/EC2 migration)
- **[`docs/vendoring.md`](docs/vendoring.md)** — `wxyc_identity_match_*` plpgsql vendoring from WXYC/wxyc-etl@v0.4.0 under `vendor/wxyc-etl/`, SHA pin in `wxyc-etl-pin.txt`, parity test (`tests/wxyc_identity_match_parity_test.rs`) enforcing pin freshness + byte-equality + 252-row PG fixture
- **[`docs/migrations.md`](docs/migrations.md)** — sqlx-cli workflow (install, add, run), idempotency requirement, dual-write convention between `migrations/000N_*.sql` and `schema/create_database.sql`, runtime vs. incremental source-of-truth split
- **[`docs/testing.md`](docs/testing.md)** — `cargo test` invocations per suite, suite breakdown (26 unit / 9 CLI / 9 oracle / 13 PG import / import integration), `TEST_DATABASE_URL` gating, build + code-style commands
- **[`docs/design-decisions.md`](docs/design-decisions.md)** — Why no SIMD (newline-delimited entities), `serde_json::from_slice` over `simd-json`, order-preserving `par_iter().map().collect()`, bounded-channel backpressure, English-only locale, entity-type priority order, `#[serde(other)]` skip behavior

Read the relevant topic doc before doing work in that area.

## Development

### TDD (Required)

All code changes follow test-driven development. No production code without a failing test first.

### Conventions

- Single-line paragraphs in commit messages and Markdown (org-wide rule).
- Use canonical WXYC artist names in test fixtures (see org-level `CLAUDE.md`); avoid Radiohead, The Beatles, Björk, etc.

## Relationship to Other Repos

- **[wxyc-etl](https://github.com/WXYC/wxyc-etl)** — Shared Rust crate this repo depends on for the `pipeline` framework, `csv_writer`, and `cli` argument groups. Canonical source of the `wxyc_identity_match_*` plpgsql functions vendored under `vendor/wxyc-etl/`.
- **[discogs-xml-converter](https://github.com/WXYC/discogs-xml-converter)** — Sibling cache-builder for the Discogs XML dump (same three-stage pipeline pattern, SIMD byte scanning instead of newline splits).
- **[musicbrainz-cache](https://github.com/WXYC/musicbrainz-cache)** — Sibling cache-builder for the MusicBrainz TSV dumps (same Rust + Postgres shape).
- **[discogs-cache](https://github.com/WXYC/discogs-etl)** — Sibling cache, runs the staggered monthly rebuild alongside this one.
- **[library-metadata-lookup](https://github.com/WXYC/library-metadata-lookup)** — Downstream consumer of the wikidata-cache Postgres DB via `DATABASE_URL_WIKIDATA` (Phase 1.5 mojibake recovery, identity resolution).
