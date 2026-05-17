# Changelog

All notable changes to `reckon-proto` will be documented in this file.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning: [SemVer](https://semver.org/) at the wire-contract level.

## [0.2.0] - 2026-05-17

### Removed (breaking) — `AdminService.ListStores`

Superseded by `StoresService.ListStores` (richer `StoreInstance`
shape with node, mode, data_dir, etc., per the cluster discovery
model). The admin variant returned a flat `repeated string store_ids`
that was conceptually a leaky abstraction and (as of the 0.1.0
extraction) collided on the message name `ListStoresRequest` /
`ListStoresResponse` with the new dedicated service.

This is a MAJOR-level wire change but cut here while no public
consumer has had time to adopt it.

#### Migration

Replace:

```
client.AdminService.ListStores(...)  → []string
```

with:

```
client.StoresService.ListStores(...) → []StoreInstance
```

The new shape carries `store_id`, `node`, `mode`, `data_dir`,
`timeout_ms`, `registered_at_us`. To recover the old flat
`store_id` list, project: `set([i.store_id for i in instances])`.

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
