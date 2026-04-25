+++
title = "Crates"
template = "section.html"
+++

Rust crates extracted from the Ferro ecosystem and published on
crates.io under Apache-2.0. Source for all of them lives in the
[`ferro-protocols`](https://github.com/youichi-uda/ferro-protocols)
mono-repo.

## Tier 1 — published

### [`ferro-lumberjack`](https://crates.io/crates/ferro-lumberjack) `v0.1.0` (beta)

Logstash Lumberjack v2 (Beats) protocol primitives — frame codec,
async client, async server, TLS via rustls, in both directions.
Extracted from production use in **FerroBeat** (Filebeat-compatible
log shipper) and **FerroHeartbeat** (Heartbeat-compatible monitor).

- [Documentation on docs.rs](https://docs.rs/ferro-lumberjack)
- [Source](https://github.com/youichi-uda/ferro-protocols/tree/main/crates/ferro-lumberjack)

### [`ferro-airflow-dag-parser`](https://crates.io/crates/ferro-airflow-dag-parser) `v0.0.1` (alpha)

Static AST-based extractor for Apache Airflow™ Python DAG files.
Recovers `dag_id`, `task_ids`, dependencies, schedule, and seven
classes of dynamic-fallback markers — without running the source.
Extracted from **FerroAir**, an Airflow-3-compatible Rust
orchestrator.

- [Documentation on docs.rs](https://docs.rs/ferro-airflow-dag-parser)
- [Source](https://github.com/youichi-uda/ferro-protocols/tree/main/crates/ferro-airflow-dag-parser)

## Roadmap (planned)

The next several crates that will land in `ferro-protocols`, in
roughly the order they ship:

- `ferro-cargo-registry-server` — Cargo Alternative Registry server-side
- `ferro-maven-layout` — Maven Repository Layout 2.0
- `ferro-oci-server` — OCI Distribution v1.1 server primitives
- `ferro-painless` — Elasticsearch Painless lexer/parser/JIT
- `ferro-esql-parser` — ES\|QL parser
- `ferro-aql-parser` — Artifactory AQL
- `ferro-pep503-pep691` — PyPI registry primitives
- `ferro-go-module-proxy` — Go module proxy server-side
- `ferro-helm-chart-repo` — Helm 3 chart repository server primitives
- `ferro-keycloak-realm-import` — Keycloak realm JSON 26.x parser/writer
- `ferro-logstash-dsl-parser` — Logstash `.conf` DSL parser

See [`docs/roadmap.md`](https://github.com/youichi-uda/ferro-protocols/blob/main/docs/roadmap.md)
for the publication schedule and Tier-2 follow-ups.

## License

All crates above are **Apache License 2.0**. Contributor flow is
[DCO](https://github.com/youichi-uda/ferro-protocols/blob/main/CONTRIBUTING.md#developer-certificate-of-origin)
(no CLA).
