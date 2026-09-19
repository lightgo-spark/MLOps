# feast-rs

**A feature store written in Rust.** Define features once in YAML, and a single binary handles point-in-time correct training dataset generation and low-latency online serving of the latest values.

No Python, no JVM, no separate server required.

## Key Features

- **Feature definitions and registry** — Validates `repo.yaml` (Entity / FeatureView / FeatureService) and produces a `registry.json` pinned by version and SHA256 hash
- **Point-in-time (PIT) joins** — Uses only the latest features at or before (`<=`) the entity timestamp, fundamentally preventing future value leakage (label leakage)
- **TTL and late-arrival handling** — Old values become NULL in training and are explicitly marked as `OUTSIDE_MAX_AGE` in serving. When timestamps tie, the value with the latest `created_timestamp` wins
- **Materialization** — Loads offline (Parquet/CSV) data into the online store. Idempotent loading so stale values never overwrite newer ones
- **Online serving** — axum-based HTTP feature server. Compatible with the Feast feature server response format (`values` / `statuses` / `event_timestamps`)
- **Online store** — Local JSON by default, Redis support via the `redis-store` feature
- **feast-studio GUI** — Desktop app (egui) for browsing catalog, lineage, statistics, quality, freshness, and serving metrics
- **feast-bench** — Benchmark suite with CSV/JSON output
- **Library** — `feast_rs` crate provides `registry`, `offline`, `online`, `materialize`, `serving`, and `profile` modules

## Components

| Binary | Role |
|---|---|
| `feast-rs` | CLI: `apply` / `get-historical` / `materialize` / `serve` |
| `feast-studio` | Desktop GUI |
| `feast-bench` | Performance benchmarking |

## Quick Start

```bash
cargo build --release

B=target/release/feast-rs
$B demo-data                    # Generate synthetic demo data
$B apply                        # Validate repo.yaml → registry v1
$B get-historical --entity-df data/entity_df.csv \
    --features "cafe_hourly_stats:*" --out data/training.parquet
$B materialize                  # Load offline → online
$B serve --port 6566            # Start feature server
```

```bash
curl -XPOST localhost:6566/get-online-features -H 'content-type: application/json' \
  -d '{"feature_service":"demand_forecast_v1","entities":{"store_id":["CAFE-001"]}}'
```

## Copyright and License

- Copyright 2026 feast-rs contributors
- The code, documentation, and images in this repository are distributed under the [Apache License 2.0](LICENSE).
- External crate dependencies follow their own licenses (MIT / Apache-2.0 / BSD, etc.), and no third-party source is included.
- The images and demo data under `docs/` are measurement and demo artifacts of this project, and all demo data is synthetic.
- Feast is a project of the LF AI & Data Foundation. This project is **not affiliated with or sponsored by** Feast or Tecton, Databricks, Amazon SageMaker, Hopsworks, etc.
