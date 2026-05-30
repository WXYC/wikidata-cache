# Observability

The binary uses `wxyc_etl::logger::init` to set up structured JSON logging on stdout and (when `SENTRY_DSN` is set) panic/error forwarding to Sentry. Every log line and Sentry event carries the four standard ETL tags:

| Tag | Value |
|-----|-------|
| `repo` | `wikidata-cache` |
| `tool` | `wikidata-cache build` or `wikidata-cache import` |
| `step` | `build` or `import` (the active subcommand) |
| `run_id` | UUIDv4 generated per process |

`SENTRY_DSN` is optional; without it, JSON logging still works and Sentry stays inactive. Provisioning the DSN in deploy environments (CI, Railway, etc.) is tracked separately.
