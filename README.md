# reckon-proto

Canonical Protocol Buffer + gRPC definitions for the [ReckonDB](https://codeberg.org/reckon-db-org/reckon-db) event store, exposed via the [reckon-gateway](https://codeberg.org/reckon-db-org/reckon-gateway) gRPC frontend.

This repo is the **single source of truth** for the wire contract. All language SDKs and the `reckon-gateway` server itself consume these proto files. Don't vendor them — depend on this repo at a pinned version.

## Services

| Service | Purpose |
|---|---|
| `StreamService` | Append + read events on streams |
| `SubscriptionService` | Live (server-streaming) and persistent subscriptions |
| `StoresService` | Cluster topology discovery (list, get, watch stores) |
| `SnapshotService` | Per-stream snapshot record/read/delete |
| `SchemaService` | Schema registration + upcasting |
| `TemporalService` | Time-travel reads |
| `CausationService` | Event-provenance DAG queries |
| `AdminService` | Store inspection, scavenge, links |
| `HealthService` | Standard gRPC health |

All per-store services accept `store_id` in every request — the proto layer is multi-store-ready. Stores are emergent (declared by deployment, discovered via `StoresService`), not API-managed.

## Layout

```
proto/
├── reckon_streams.proto
├── reckon_subscriptions.proto
├── reckon_stores.proto
├── reckon_snapshots.proto
├── reckon_schema.proto
├── reckon_temporal.proto
├── reckon_causation.proto
├── reckon_admin.proto
├── reckon_health.proto
└── reckon_shared.proto
```

Package: `reckon.gateway.v1` for every service.

## Codegen

Per-language stubs generated via [buf](https://buf.build):

```bash
buf generate
```

Targets are declared in `buf.gen.yaml`. Currently shipping:

- **Go** — `codeberg.org/reckon-db-org/reckon-go` (separate repo)

Coming as those SDKs land: TypeScript, Rust, C#, Kotlin.

## Versioning

`reckon-proto` follows SemVer at the wire-contract level:

- **MAJOR** — breaking wire change (field removed, type changed, RPC removed)
- **MINOR** — additive (new field, new RPC, new service) — old clients still work
- **PATCH** — comment-only or layout changes, no wire-level effect

SDKs and the gateway pin to a specific minor or higher (`~> 0.1`).

## Status

`v0.1.0` — initial extraction from `reckon-gateway/proto/`. Wire-compatible with `reckon-gateway 0.4.x` and `reckon-db 2.2.x`.

## License

Apache-2.0.
