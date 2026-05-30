# Architecture

## Modules

- `model.rs` -- Data structures for Wikidata JSON entities. Only fields needed for filtering and extraction are modeled; everything else is skipped during deserialization via `serde`. Key types: `Entity`, `Statement`, `Snak`, `DataValue`.
- `filter.rs` -- Music-relevance filter. Primary indicators: P1953 (Discogs artist ID), P1902 (Spotify artist ID), P106 (musician occupation), P31 (musical group / record label). Secondary properties (P737, P136, P264, P749, P2850, P3283) are extracted but don't independently qualify entities.
- `extractor.rs` -- Extracts flat CSV rows from matched entities. Classifies entity type (human/group/label/other) from P31/P106 claims. Produces rows for 8 output tables. Extracts external IDs (P1953 Discogs, P434 MusicBrainz, P1902 Spotify, P2850 Apple Music, P3283 Bandcamp) into `discogs_mapping.csv`.
- `writer.rs` -- `CsvOutput` wraps `wxyc_etl::csv_writer::MultiCsvWriter` for 8 CSV files with headers matching the wikidata-cache PostgreSQL schema. Implements `wxyc_etl::pipeline::PipelineOutput<ExtractedRows>`. The `csv_file_specs()` function defines the 8-file spec.
- `import.rs` -- CSV import module. Reads the 8 CSV files and streams them into PostgreSQL via COPY TEXT. Handles RFC 4180 quoted fields, Unicode, and empty CSVs.
- `import_schema.rs` -- PostgreSQL schema management. Embeds and applies `schema/create_database.sql`. Provides UNLOGGED/LOGGED toggle and VACUUM FULL for bulk import performance. Table constants define FK-safe import order.
- `main.rs` -- CLI (clap derive) using shared argument groups from `wxyc_etl::cli` (`DatabaseArgs`, `ResumableBuildArgs`, `ImportArgs`). The `build` subcommand runs the three-stage filter pipeline via `wxyc_etl::pipeline`; the `import` subcommand loads CSVs into PostgreSQL. `--database-url` falls back to `DATABASE_URL_WIKIDATA` via `wxyc_etl::cli::resolve_database_url`. `--output-dir` (build) and `--csv-dir` (import) are accepted as deprecated aliases for `--data-dir` with a stderr warning. Initializes `wxyc_etl::logger` (Sentry + JSON logs) at startup and wraps each subcommand in a tracing span tagged `repo`/`tool`/`step`.

## Parallel Processing Pipeline

Uses `wxyc_etl::pipeline` framework (same three-stage pattern as discogs-xml-converter):

1. **Scanner thread** (`start_scanner`) -- reads the input (gzipped or plain) via `flate2::GzDecoder` + `BufReader`, reads line by line (the Wikidata dump is `[\n{entity},\n{entity},\n...\n]`), strips array brackets and trailing commas, sends raw byte vectors via `BatchSender`. Batch size and channel capacity use `BatchConfig::default()` (256 items, 64 batches).
2. **Rayon worker pool** (`run_pipeline`) -- receives batches, deserializes JSON via `serde_json::from_slice`, applies music-relevance filter, extracts target fields from matched entities. Preserves input order.
3. **Writer** (`PipelineOutput`) -- `CsvOutput` writes extracted rows to 8 CSV files in document order.

No SIMD byte scanning needed (unlike discogs-xml-converter) because entity boundaries are newlines.

## CSV Output Contract

The 8 output CSV files must be compatible with `wikidata-cache/scripts/import_csv.py`. Headers and column order are defined in `writer.rs`. Changes to the CSV schema require coordinating with wikidata-cache.

| File | Columns |
|------|---------|
| `entity.csv` | qid, label, description, entity_type |
| `discogs_mapping.csv` | qid, property, discogs_id |
| `influence.csv` | source_qid, target_qid |
| `genre.csv` | entity_qid, genre_qid |
| `record_label.csv` | artist_qid, label_qid |
| `label_hierarchy.csv` | child_qid, parent_qid |
| `entity_alias.csv` | qid, alias |
| `occupation.csv` | entity_qid, occupation_qid |

## PostgreSQL Schema

Defined in `schema/create_database.sql` and embedded via `include_str!` in `import_schema.rs`. The schema uses pg_trgm for trigram indexes on `entity.label` and `entity_alias.alias` for fuzzy text search. FK constraints enforce referential integrity from child tables to `entity.qid`, except `influence.target_qid` and `label_hierarchy` which allow dangling references (the target entity may have been filtered out).

## Filter Criteria

An entity is music-relevant if it has ANY primary indicator (each sufficient on its own):
- **P1953** (Discogs artist ID) -- entity has a Discogs page
- **P1902** (Discogs label ID) -- record label with a Discogs page
- **P106** (occupation) with a musician-related QID. The full set is defined in `filter.rs::MUSICIAN_QIDS`.
- **P31** (instance of) with a musical group or record label QID. The full set is defined in `filter.rs::MUSICAL_GROUP_QIDS`.

Secondary properties (P737 influence, P136 genre, P264 record label, P749 parent org, P2850 Apple Music artist ID, P3283 Bandcamp profile ID) are extracted only from entities that pass the primary filter. They don't independently qualify an entity.
