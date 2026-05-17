# Changelog

All notable changes to `reckon-proto` will be documented in this file.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning: [SemVer](https://semver.org/) at the wire-contract level.

## [0.1.0] - 2026-05-17

### Added — Initial extraction

Canonical proto bundle extracted from `reckon-gateway/proto/`.
Wire-compatible with `reckon-gateway 0.4.0` and `reckon-db 2.2.0`.

Services:

- `StreamService` — append + read events
- `SubscriptionService` — live (server-streaming) + persistent subscriptions
- `StoresService` — cluster topology discovery
- `SnapshotService` — per-stream snapshots
- `SchemaService` — schema registration + upcasting
- `TemporalService` — time-travel reads
- `CausationService` — event-provenance DAG
- `AdminService` — store inspection, scavenge, links
- `HealthService` — standard gRPC health

`go_package` repointed from the embedded gateway path to
`codeberg.org/reckon-db-org/reckon-go/genproto/gatewayv1`, where the Go
SDK will consume the generated bindings.
