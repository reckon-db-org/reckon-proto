# reckon-proto
[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-support-yellow.svg)](https://buymeacoffee.com/rlefever)

Canonical Protocol Buffer + gRPC definitions for the [ReckonDB](https://github.com/reckon-db-org/reckon-db) event store, exposed via the [reckon-gateway](https://github.com/reckon-db-org/reckon-gateway) gRPC frontend.

This repo is the **single source of truth** for the wire contract. All language SDKs and the `reckon-gateway` server itself consume these proto files. Don't vendor them; depend on this repo at a pinned version.

Package: `reckon.gateway.v1` for every service. `proto3` syntax. Every per-store RPC carries `store_id` in its request, so the proto layer is multi-store-ready. Stores are emergent (declared by deployment, discovered via `StoresService`), not API-managed.

## Services

Nine services across ten `.proto` files. Every RPC lives in package `reckon.gateway.v1`.

| Service | Proto file | Key RPCs |
|---|---|---|
| `StreamService` | `reckon_streams.proto` | `AppendEvents`, `ReadStreamForward`, `ReadStreamBackward`, `StreamEventsForward` (server-stream), `GetStreamVersion`, `ListStreams`, `DeleteStream`, `ReadByEventTypes`, `ReadByTags`, `ReadByMetadata`, `ReadAllGlobal` |
| `SubscriptionService` | `reckon_subscriptions.proto` | `Subscribe` (server-stream), `AckEvent`, `CreateSubscription`, `RemoveSubscription`, `ListSubscriptions`, `GetSubscription`, `GetSubscriptionLag` |
| `StoresService` | `reckon_stores.proto` | `ListStores`, `GetStore`, `WatchStores` (server-stream) |
| `SnapshotService` | `reckon_snapshots.proto` | `RecordSnapshot`, `ReadSnapshot`, `DeleteSnapshot`, `ListSnapshots`, `ListAllSnapshots` |
| `SchemaService` | `reckon_schema.proto` | `RegisterSchema`, `UnregisterSchema`, `GetSchema`, `ListSchemas`, `GetSchemaVersion`, `UpcastEvents` |
| `TemporalService` | `reckon_temporal.proto` | `ReadUntil`, `ReadRange`, `VersionAt` |
| `AdminService` | `reckon_admin.proto` | `GetStoreStats`, `GetStreamInfo`, `GetEventTypeSummary`, `Scavenge`, `ScavengeMatching`, `ScavengeDryRun`, `CreateLink`, `DeleteLink`, `GetLink`, `ListLinks`, `StartLink`, `StopLink`, `GetLinkInfo`, `ReloadCatalogue`, `GetCatalogueStatus` |
| `HealthService` | `reckon_health.proto` | `Check`, `Health`, `VerifyClusterConsistency`, `VerifyMembershipConsensus`, `CheckRaftLogConsistency`, `GetMemoryLevel`, `GetMemoryStats`, `GetServerInfo` |
| `DcbService` | `reckon_dcb.proto` | `AppendIfNoTagMatches`, `ReadDcbContext`, `CccReadByPayload`, `CccReadByPayloadHash` |

### DcbService: Dynamic Consistency Boundary + CCC

`DcbService` is the cross-cutting complement to per-stream optimistic concurrency. Where `StreamService.AppendEvents` enforces consistency against a per-stream version counter, DCB enforces it against a tag-filter context query: "no event matching this filter has been written since I last looked". Use it when the invariant a write must preserve crosses streams (uniqueness, allocation, rate-limit, eligibility) rather than living inside one aggregate.

- `AppendIfNoTagMatches`: conditional append under the DCB pseudo-stream. The server rejects the write if any event matching the request's `TagFilter` has seq strictly above `seq_cutoff`. The conflict comes back as a structured `oneof { Committed | Conflict }` (a successful response shape, not a gRPC error); gRPC status codes stay reserved for transport / backend errors.
- `ReadDcbContext`: read events matching a `TagFilter` from the DCB pseudo-stream, seq-ascending, with the highest observed seq returned alongside so the caller can compute the `seq_cutoff` for a follow-up append.
- `CccReadByPayload` / `CccReadByPayloadHash`: CCC (Command Context Consistency) payload-indexed reads. They find events by JSON `data` field values (single key, or a combo hash over an ordered key set) using the store's CCC indexes, without tag matching. They are the read half of a CCC decision loop where the consistency predicate is over payload content rather than tags.

`TagFilter` is a recursive predicate with three leaf shapes (`match_any`, `match_all`, `event_type_match`) and two compound shapes (`conjunction`, `disjunction`). The `event_type_match` leaf scopes a context by event type and is backed by the `[by_event_type]` Khepri index (reckon-db 5.2.0+). CCC payload reads require a `{ccc, key}` or `{ccc_hash, keys}` index declared in the store config (reckon-db 5.4.0+). Pre-DCB backing clusters surface as gRPC `UNIMPLEMENTED`. See [the DCB guide](https://github.com/reckon-db-org/reckon-db/blob/main/guides/dcb.md).

### Shared messages

`reckon_shared.proto` holds the cross-service message types imported by the others:

- `ProposedEvent`: an event to be appended (`event_type`, `data`, `metadata`, optional tags).
- `RecordedEvent`: an event read back from the store (adds assigned ids, stream position, timestamps).
- `SnapshotRecord`: a captured aggregate-state snapshot.
- `SubscriptionInfo`: persistent-subscription descriptor.

`reckon_shared.proto` also pins the **reserved metadata keys** (`causation_id`, `correlation_id`, `conversation_id`) as the cross-language JSON keys inside an event's opaque `metadata`. That comment block is the single source of truth from which every client library defines its constants. The store stores and returns metadata verbatim; it never interprets it. Lineage traversal is a read-model concern, queried via `StreamService.ReadByMetadata`; there is deliberately no server-side causation-graph verb.

## Layout

```
proto/
├── reckon_streams.proto
├── reckon_subscriptions.proto
├── reckon_stores.proto
├── reckon_snapshots.proto
├── reckon_schema.proto
├── reckon_temporal.proto
├── reckon_admin.proto
├── reckon_dcb.proto
├── reckon_health.proto
└── reckon_shared.proto
```

## How consumers fetch it

This OTP app carries no Erlang behaviour. It exists so consumers can pull the proto bundle two ways from one source:

- **Erlang / rebar3 (reckon-gateway)**: declare `reckon_proto` as a git dep and point `grpc_plugin` at `proto/` inside the fetched dep dir. Stubs regenerate on `rebar3 compile`; the gateway never vendors a copy.
- **Polyglot SDKs (reckon-go, future)**: ignore the OTP metadata and consume `proto/` directly via [buf](https://buf.build).

### Codegen

Per-language stubs are generated via buf. Targets are declared in `buf.gen.yaml`; `buf.yaml` declares the module and lint/breaking rules:

```bash
buf generate
```

Currently shipping:

- **Go**: [reckon-go](https://github.com/reckon-db-org/reckon-go) (`genproto/gatewayv1`, separate repo)

Coming as those SDKs land: TypeScript, Rust, C#, Kotlin.

## Versioning

`reckon-proto` follows SemVer at the wire-contract level:

- **MAJOR**: breaking wire change (field removed, type changed, RPC removed)
- **MINOR**: additive (new field, new RPC, new service); old clients still work
- **PATCH**: comment-only or layout changes, no wire-level effect

SDKs and the gateway pin to a specific minor or higher (`~> 0.7`).

## Status

`v0.7.0` (current). The surface grew well past the initial extraction: `DcbService` (DCB conditional append + context read), the `TagFilter.event_type_match` leaf, and the CCC payload-indexed reads (`CccReadByPayload` / `CccReadByPayloadHash`) are all part of the contract now. See [CHANGELOG.md](CHANGELOG.md) for the full history. Wire-compatible with current `reckon-gateway` and `reckon-db 5.x` backings.

## Reckon stack

reckon-proto is one library in the Reckon event-sourcing ecosystem. In dependency order (a library only knows about the ones above it):

- **reckon-proto (this repo)**: the wire-contract protobufs; source of truth for the gateway surface. Consumed at build time by reckon-gateway and reckon-go.
- **[reckon-gater](https://github.com/reckon-db-org/reckon-gater)**: shared Erlang types and protocols; no Reckon dependencies.
- **[reckon-db](https://github.com/reckon-db-org/reckon-db)**: BEAM-native event store. Depends on reckon_gater, khepri, ra.
- **[reckon-nifs](https://github.com/reckon-db-org/reckon-nifs)**: standalone Rust NIF helpers with pure-Erlang fallbacks.
- **[evoq](https://github.com/reckon-db-org/evoq)**: standalone CQRS/event-sourcing framework; no Reckon dependencies.
- **[reckon-evoq](https://github.com/reckon-db-org/reckon-evoq)**: adapter wiring evoq to a Reckon store. Depends on evoq and reckon_gater; not on reckon_db.
- **[reckon-gateway](https://github.com/reckon-db-org/reckon-gateway)**: gRPC + HTTP/JSON ingress that serves this proto surface. Consumes reckon_gater; can embed reckon_db or federate remote clusters.
- **[reckon-go](https://github.com/reckon-db-org/reckon-go)**: the Go client generated against these protos; talks to reckon-gateway.
- **reckon-portal**: docs and landing site ([reckon-internal/reckon-portal](https://github.com/reckon-db-org/reckon-portal)).

## License

Apache-2.0.
</content>
</invoke>
