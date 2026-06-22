# Changelog

All notable changes to `reckon-proto` will be documented in this file.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning: [SemVer](https://semver.org/) at the wire-contract level.

## [0.7.0] - 2026-06-22

### Added — `TagFilter.event_type_match` (field 5)

`reckon_dcb.proto`: `TagFilter.oneof kind` gains a fifth leaf shape:

```proto
string event_type_match = 5;
```

Maps to `{event_type, binary()}` in `reckon_gater_types:tag_filter()`.

Clients can now scope a DCB consistency context by event type rather than
(or in addition to) tags, e.g.:

```
TagFilter { event_type_match: "inventory_reserved_v1" }
```

Backed by the `[by_event_type]` Khepri index added in reckon-db 5.2.0.
Pre-5.2.0 events have no index entry and will not match.

Wire-backwards-compatible: older clients that do not set field 5 see no
change. Older servers (pre-5.2.0 gateway) will return a validation error
for requests that set `event_type_match`.

## [0.6.1] - 2026-06-08

### Added — reserved metadata-key contract (docs only, no wire change)

Documented `causation_id` / `correlation_id` / `conversation_id` as the
**reserved, cross-language metadata JSON keys** in `reckon_shared.proto`,
pinning the Enterprise Integration Patterns correlation/causation
identifiers at the wire-contract level so every client (BEAM/evoq, Go, …)
agrees on the names. This comment block is the single source of truth from
which each client library defines its constants.

No message, field, or RPC change — `metadata` remains opaque JSON bytes.
The keys are queried via the existing `StreamService.ReadByMetadata`; there
is deliberately no server-side `get_effects`/`get_causes`/graph verb
(lineage is a read-model concern, mirroring EventStoreDB's
`$by_correlation_id` projection). Propagation is a producing-framework
responsibility (evoq on BEAM); raw clients set the keys themselves.

## [0.6.0] - 2026-06-08

### Added — `StreamService.ReadByMetadata`

New cross-cutting read RPC alongside `ReadByTags` / `ReadByEventTypes`:

```proto
rpc ReadByMetadata(ReadByMetadataRequest) returns (ReadStreamResponse);

message ReadByMetadataRequest {
  string store_id = 1;
  string key = 2;
  string value = 3;
  uint64 batch_size = 4;
}
```

Reads events whose metadata `key = value`. The sanctioned primitive for
application-built causation/correlation read models — O(matches) when the
store declared the `{meta, key}` secondary index (reckon-db 5.0.0+),
otherwise a server-side scan. The store does not interpret the key.

Additive and backward-compatible. Surfaces reckon-db 5.0.0 /
reckon_gater 3.2.0 `read_by_metadata`. Downstream: reckon-gateway adds a
handler; reckon-go regenerates stubs and adds a `ReadByMetadata` method.

## [0.5.0] - 2026-06-07

### Removed — CausationService (BREAKING)

Deleted `proto/reckon_causation.proto` (`CausationService` and its
`Causation*` / `Correlation*` messages: `GetEffects`, `GetCause`,
`GetCausationChain`, `GetCorrelated`, `BuildCausationGraph`).

Causation/correlation lineage is **not an event-store concern**.
`causation_id` and `correlation_id` remain exactly what they always
were — opaque keys inside an event's `metadata` (see the note on
`RecordedEvent.metadata` in `reckon_shared.proto`). The producer sets
them; consumers that need to *traverse* lineage build a read model /
projection (or ship it to tracing), the same way EventStoreDB serves
causation via system projections rather than scanning the log. The
store stores and returns metadata verbatim; it does not interpret it.

## [0.4.0] - 2026-05-27

### Added — DCB service for polyglot conditional-append

New file `proto/reckon_dcb.proto` exposing the Dynamic Consistency
Boundary primitive of ReckonDB (paired with reckon-db 3.1.1+,
reckon-gater 2.3.1+) to gRPC clients.

`DcbService`:

- `AppendIfNoTagMatches(AppendIfNoTagMatchesRequest)` — conditional
  append under the DCB pseudo-stream. Server rejects when any
  event matching the request's `tag_filter` has seq strictly above
  `seq_cutoff`. Returns a structured `oneof { Committed | Conflict }`
  so the conflict path is a successful response shape, not a gRPC
  error. gRPC status codes stay reserved for transport / backend
  errors (`UNAVAILABLE`, `UNIMPLEMENTED`, `INTERNAL`).
- `ReadDcbContext(ReadDcbContextRequest)` — read events matching a
  `TagFilter` from the DCB pseudo-stream, ordered by seq ascending,
  with the highest seq returned alongside. Use this to compute the
  `seq_cutoff` for a subsequent `AppendIfNoTagMatches`.

`TagFilter` is a recursive `oneof` with four variants:

```protobuf
message TagFilter {
  oneof kind {
    TagList    match_any   = 1;  // {any_of,  [Tag]}
    TagList    match_all   = 2;  // {all_of,  [Tag]}
    FilterList conjunction = 3;  // {and_,    [TagFilter]}
    FilterList disjunction = 4;  // {or_,     [TagFilter]}
  }
}
```

Variant names avoid the `and_` / `or_` reserved-word collision in
polyglot consumers while preserving the Erlang term mapping 1:1.

`seq_cutoff` is `sint64` (ZigZag-encoded) so the `-1` "saw nothing"
sentinel is a single byte on the wire.

Pre-DCB backing clusters surface as gRPC `UNIMPLEMENTED`; the
gateway does not carry a partial-support path.

### Notes

Additive only. Existing `StreamService`, `SubscriptionService`,
etc., are untouched. Consumers regenerate bindings against 0.4.0
to pick up the new service.

Design doc: `reckon-gateway/plans/DESIGN_DCB_GRPC_SURFACE.md`.

## [0.3.1] - 2026-05-20

### Fixed (breaking-on-wire-but-server-shape-stable) — rename `ClusterStatus` to `CatalogueClusterStatus` in `reckon_admin.proto`

v0.3.0 introduced `message ClusterStatus` in `reckon_admin.proto`, but
`reckon_health.proto` already defined `enum ClusterStatus` in the same
`reckon.gateway.v1` namespace. Erlang's `gpb` compiler tolerates this
(per-file module separation) so reckon-gateway builds fine, but `buf`
and `protoc` reject it as a duplicate symbol — meaning every non-Erlang
consumer (reckon-go and any future polyglot SDK) cannot regenerate
stubs from v0.3.0.

Rename the new message to `CatalogueClusterStatus` (more descriptive
anyway — it describes catalogue connector status, not raft quorum).
Field tags and shape unchanged; wire format identical for any client
that maps the response into its own type. Erlang server-side code in
`reckon_gateway_admin_service:cluster_to_proto/1` is shape-based and
needs no edit (the on-the-wire bytes are the same).

## [0.3.0] - 2026-05-19

### Added — `AdminService` catalogue RPCs

Two new RPCs on `AdminService`, for use by catalogue-mode
reckon-gateway (0.5+):

  - `ReloadCatalogue(ReloadCatalogueRequest)` → re-reads
    operator-curated `clusters.eterm`, reconciles running cluster
    connectors. Response carries the diff: `added`, `removed`,
    `restarted` cluster_ids, plus an `error` string populated only
    on config-load failure (no state mutation in that case).
  - `GetCatalogueStatus(GetCatalogueStatusRequest)` → read-only
    snapshot. Returns `catalogue_size`, `gateway_uptime_ms`, and a
    list of `ClusterStatus` entries (cluster_id, members, store
    count, status, last_refresh, last_error).

Cookies are deliberately NEVER returned by either RPC. The
operator's `clusters.eterm` is the only on-disk copy.

Non-breaking: existing AdminService methods are unchanged.

### Updated bindings

Erlang gRPC stubs in reckon-gateway are regenerated by
`grpc_plugin` from this proto on rebar3 compile. Go stubs in
reckon-go follow the existing buf workflow (`buf generate`).

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
