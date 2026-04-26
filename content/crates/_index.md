+++
title = "Crates"
template = "section.html"
+++

Rust crates extracted from the Ferro ecosystem and published on
crates.io under Apache-2.0. Source for all of them lives in the
[`ferro-protocols`](https://github.com/youichi-uda/ferro-protocols)
mono-repo.

## Tier 1 — published

### [`ferro-blob-store`](https://crates.io/crates/ferro-blob-store) `v0.0.3` (alpha)

Foundation crate for content-addressed blob storage. Five-method
async `BlobStore` trait plus `InMemoryBlobStore` and (default
feature) `FsBlobStore` reference backends. Used as the storage
abstraction under the OCI / Maven / Cargo crates below.

- [Documentation on docs.rs](https://docs.rs/ferro-blob-store)
- [Source](https://github.com/youichi-uda/ferro-protocols/tree/main/crates/ferro-blob-store)

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

### [`ferro-maven-layout`](https://crates.io/crates/ferro-maven-layout) `v0.0.1` (alpha)

Apache Maven™ Repository Layout 2.0 — GAV parsing,
`maven-metadata.xml`, minimal POM parser, SNAPSHOT timestamp +
buildNumber, SHA-1/SHA-256 (and `legacy-md5`) checksum helpers, plus
an Axum HTTP router that mounts a `BlobStore` from
`ferro-blob-store` at any URL prefix.

- [Documentation on docs.rs](https://docs.rs/ferro-maven-layout)
- [Source](https://github.com/youichi-uda/ferro-protocols/tree/main/crates/ferro-maven-layout)

### [`ferro-cargo-registry-server`](https://crates.io/crates/ferro-cargo-registry-server) `v0.0.1` (alpha)

**Server-side** primitives for the Cargo Alternative Registry
Protocol (sparse index variant). As far as we can tell, the first
crate on crates.io that publishes the server half of this
protocol. `/config.json`, sparse index entries,
publish / yank / unyank / owners API, and an Axum router.

- [Documentation on docs.rs](https://docs.rs/ferro-cargo-registry-server)
- [Source](https://github.com/youichi-uda/ferro-protocols/tree/main/crates/ferro-cargo-registry-server)

### [`ferro-oci-server`](https://crates.io/crates/ferro-oci-server) `v0.0.1` (alpha)

**Server-side** primitives for the OCI Distribution Specification
v1.1 — manifest / blob / tag / referrers handlers, chunked-upload
state machine, error envelope per spec §6.2, and an in-memory
`RegistryMeta` reference impl. Existing Rust crates implement the
client (`oci-client`) and types (`oci-spec`); this is the first
server-side primitive crate. The `v0.1.0` gate is passing the
upstream `opencontainers/distribution-spec` conformance suite.

- [Documentation on docs.rs](https://docs.rs/ferro-oci-server)
- [Source](https://github.com/youichi-uda/ferro-protocols/tree/main/crates/ferro-oci-server)

## Tier 2 — planned

The next several crates that will land in `ferro-protocols`, in
roughly the order they ship:

- `ferro-pep503-pep691` — PyPI registry primitives
- `ferro-go-module-proxy` — Go module proxy server-side
- `ferro-helm-chart-repo` — Helm 3 chart repository server primitives
- `ferro-painless` — Elasticsearch Painless lexer/parser/JIT
- `ferro-esql-parser` — ES\|QL parser
- `ferro-aql-parser` — Artifactory AQL
- `ferro-keycloak-realm-import` — Keycloak realm JSON 26.x parser/writer
- `ferro-logstash-dsl-parser` — Logstash `.conf` DSL parser

See [`docs/roadmap.md`](https://github.com/youichi-uda/ferro-protocols/blob/main/docs/roadmap.md)
for the full schedule.

## License

All crates above are **Apache License 2.0**. Contributor flow is
[DCO](https://github.com/youichi-uda/ferro-protocols/blob/main/CONTRIBUTING.md#developer-certificate-of-origin)
(no CLA).
