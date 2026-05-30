# Key Design Decisions

- Entity boundaries are newlines, so no byte scanning is needed (unlike discogs-xml-converter's SIMD `memchr` approach for XML element boundaries)
- `serde_json::from_slice` is used for JSON parsing; `simd-json` is a potential optimization if parsing becomes the bottleneck
- `par_iter().map().collect()` preserves input order so CSV output is deterministic regardless of thread scheduling
- Bounded channel (capacity 64 batches of 256 entities) provides backpressure to prevent unbounded memory growth
- Only English labels, descriptions, and aliases are extracted (`labels.en`, `descriptions.en`, `aliases.en`)
- Entity type classification priority: record label > musical group > human > other (checked in that order from P31/P106)
- The `DataValue` enum uses `#[serde(other)]` to silently skip unknown value types (time, quantity, coordinates, etc.) without failing deserialization
