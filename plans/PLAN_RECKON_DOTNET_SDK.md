# PLAN — reckon-dotnet (.NET gRPC SDK)

**Status:** Draft, 2026-07-01
**Owner:** reckon-db-org
**Scope:** A typed, idiomatic .NET client for ReckonDB over the reckon-proto
gRPC contract, reached through reckon-gateway. Mirrors reckon-go 1:1 in shape.
**Non-goal:** No .NET port of evoq (the framework layer). Decision recorded in
the Reckon Codex appendix "Reckon and the Critter Stack". The framework stays on
the BEAM; .NET teams keep Wolverine/Marten idioms and back the *store* with
ReckonDB via this SDK.

---

## 1. Positioning

Thin typed façade over `reckon.gateway.v1`, exactly parallel to reckon-go:
**one channel per gateway endpoint, per-service sub-clients bound to a store.**
No Erlang, no framework port. Consumers `dotnet add package Reckon.Client` and
get precompiled assemblies — they never touch protoc or buf.

- New repo: `codeberg.org/reckon-db-org/reckon-dotnet`.
- Package id `Reckon.Client`, root namespace `Reckon` (calls read
  `ReckonClient.Connect(...)`, matching Go's `reckon.Connect`).

---

## 2. Codegen strategy

Use **`Grpc.Tools`** (native .NET path), not buf, for the SDK build:

- reckon-proto vendored as a **git submodule** pinned to a tag. Canonical source
  stays reckon-proto; zero drift.
- `.csproj` references the `.proto` via
  `<Protobuf Include="proto/*.proto" GrpcServices="Client" />`. Grpc.Tools runs
  protoc at **SDK build time** into `obj/`; generated C# is never committed.
- Ship the compiled NuGet. Consumers get typed stubs with no protoc/buf
  dependency. This is the .NET idiom (how every gRPC .NET lib ships).

Fallback for parity with Go's generation flow: add
`buf.build/protocolbuffers/csharp` + `buf.build/grpc/csharp` targets to
reckon-proto's `buf.gen.yaml`. Kept as an option; Grpc.Tools gives better
MSBuild integration and is preferred.

---

## 3. Repository layout (mirrors reckon-go's `*_facade.go`)

```
reckon-dotnet/
├── proto/                         # git submodule → reckon-proto @ tag
├── src/
│   ├── Reckon.Client/             # THE package
│   │   ├── ReckonClient.cs        # facade: channel + Connect/Close + sub-client factories
│   │   ├── ReckonClientOptions.cs # TLS / Insecure / CA / serverName / interceptors
│   │   ├── Streams/               # StreamsClient, ProposedEvent, RecordedEvent, StreamState
│   │   ├── Subscriptions/         # SubscriptionsClient (IAsyncEnumerable + Ack + persistent)
│   │   ├── Dcb/                   # DcbClient (DCB + CCC reads), DcbFilter, DcbAppendResult
│   │   ├── Snapshots/  Schema/  Temporal/  Admin/  Stores/  Health/
│   │   └── Reckon.Client.csproj   # <Protobuf Include> + Grpc.Net.Client + Grpc.Tools
│   └── Reckon.Extensions.Hosting/ # optional: DI + subscription worker base (§7)
├── test/
│   ├── Reckon.Client.Tests/       # xUnit unit tests
│   └── Reckon.Client.E2E/         # against lab gateway beam01.lab:50051
├── examples/                      # streams, subscriptions, dcb, admin (mirror reckon-go/examples)
└── plans/
```

- **Layer 1 (generated):** `Reckon.Gateway.V1.*` — build-time stubs.
- **Layer 2 (façade):** hand-written `Reckon.*` — the public surface. Wraps
  stubs, maps to records, hides protobuf.
- **Layer 3 (hosting):** DI + `BackgroundService` glue. Optional package.

---

## 4. Sub-client map (1:1 with reckon-go's 9 services)

| reckon-go | reckon-dotnet |
|---|---|
| `c.Stores()` | `client.Stores` |
| `c.Streams(store)` | `client.Streams(store)` |
| `c.Subscriptions(store)` | `client.Subscriptions(store)` |
| `c.Snapshots(store)` | `client.Snapshots(store)` |
| `c.Dcb(store)` | `client.Dcb(store)` |
| `c.Schema(store)` | `client.Schema(store)` |
| `c.Temporal(store)` | `client.Temporal(store)` |
| `c.Admin(store)` | `client.Admin(store)` |
| `c.Health()` | `client.Health` |

Store-bound sub-clients are cheap structs reusing the one channel;
`Stores`/`Health` are gateway-wide.

**Note — nine services, not ten primitives.** There is no `Ccc` sub-client.
CCC (Command Context Consistency) is exposed as payload-keyed *read* methods on
the **Dcb** sub-client, mirroring the proto (`CccReadByPayload` /
`CccReadByPayloadHash` live inside `DcbService`) and reckon-go
(`dcb/ccc.go` on `c.Dcb(store)`). Rationale in §5.

---

## 5. Why Dcb has a dedicated service/space but Ccc does not

Short answer: **CCC is not a separate primitive; it is the payload-keyed read
variant of the DCB primitive, and it has no write of its own.**

Longer:

1. **One primitive, one boundary.** DCB is the consistency mechanism: enforce an
   invariant against a *context query* ("no event matching this predicate has
   been written since I last looked") instead of a per-stream version counter.
   CCC changes only the *predicate shape* — payload fields (`data[key] == value`)
   instead of tags. Same boundary, same read-decide-append loop, different leaf.

2. **CCC has no append.** There is exactly one conditional-append RPC in the
   whole space: `DcbService.AppendIfNoTagMatches`. CCC adds only alternate
   *reads* (`CccReadByPayload`, `CccReadByPayloadHash`) that feed that same
   append. A `CccService` would either duplicate the DCB append or ship a
   read-only service — both are worse than co-locating the reads with the write
   they serve.

3. **A gRPC service is a cohesive RPC group around one primitive.** DCB is that
   primitive; CCC is its payload-conditioned refinement. Splitting them would
   fragment a single read-decide-append loop across two services and imply a
   symmetry (CCC-with-its-own-append) that does not exist.

4. **The stack is consistent top to bottom.** evoq treats them the same way:
   one `evoq_decision` behaviour, with CCC expressed as `payload_match` /
   `payload_hash_match` leaves in the same decision/filter model — not a
   separate behaviour. reckon-gater exposes `ccc_read_by_payload/4` alongside
   the DCB `append_if_no_tag_matches/4` on the same API surface. reckon-proto
   folds the CCC reads into `DcbService`. reckon-go puts them on the `dcb`
   sub-client. reckon-dotnet does the same. CCC is a *feature of DCB*, and every
   layer names it that way.

So the SDK's `DcbClient` carries both: `ReadAsync`/`AppendAsync` (tag DCB) and
`CccReadByPayloadAsync`/`CccReadByPayloadHashAsync` (payload CCC reads that feed
the same `AppendAsync`).

---

## 6. Idiomatic .NET translations

| reckon-go | reckon-dotnet |
|---|---|
| `Connect(ctx, addr, opts...)` | `ReckonClient.Connect(addr, options)` / `ConnectAsync` |
| `Insecure()`, `TLSWithCA(...)` | `ReckonClientOptions { Insecure, CaCertificatePath, ServerNameOverride }` (default = TLS, system roots) |
| `context.Context` | `CancellationToken` on every call |
| return `(res, err)` | `Task<T>` / `ValueTask<T>`, exceptions for transport errors |
| `Watch`/`Subscribe` → channel + errchan | **`IAsyncEnumerable<RecordedEvent>`** (`await foreach`) |
| `AnyVersion`/`NoStream`/`StreamExists` + int | **`StreamState`** readonly struct: `.Any`, `.NoStream`, `.StreamExists`, `.AtVersion(long)` — deliberately mirrors EventStoreDB's .NET client so ES devs feel at home |
| `ProposedEvent{EventType, Data, Tags}` | `record ProposedEvent(string EventType, ReadOnlyMemory<byte> Data, ...)` |
| DCB `(committed, conflict, err)` | `DcbAppendResult` record with `Committed?` / `Conflict?` (conflict is control flow, **not** an exception) |

Everything `async`, `nullable` enabled, target **`net8.0`** (LTS) + `net9.0`.

---

## 7. Façade shape (public surface)

```csharp
// Connect — TLS by default, system roots
await using var client = await ReckonClient.ConnectAsync(
    "beam01.lab:50051", new ReckonClientOptions { Insecure = true });

// Append with optimistic concurrency
var streams = client.Streams("default_store");
var res = await streams.AppendAsync("users-42", StreamState.Any, new[]
{
    new ProposedEvent("user_registered_v1", """{"name":"Ada"}"""u8.ToArray()),
});
// res => AppendResult(Version, Position, Count)

// Read forward
await foreach (var e in streams.ReadAsync("users-42", fromVersion: 0, maxCount: 100, ct))
    Console.WriteLine($"v={e.Version} {e.EventType}");

// DCB read-decide-append loop
var dcb = client.Dcb("orders");
var context = await dcb.ReadAsync(DcbFilter.MatchAny("slot:42"), 100, ct);
var outcome = await dcb.AppendAsync(DcbFilter.MatchAny("slot:42"), context.MaxSeq,
    new[] { new ProposedEvent("slot_reserved_v1", tags: new[] { "slot:42" }) }, ct);
if (outcome.Conflict is not null) { /* context stale, retry */ }

// CCC — payload-keyed read on the SAME Dcb sub-client, feeds the same AppendAsync
var byAccount = await dcb.CccReadByPayloadAsync("account_id", "acc-42", 100, ct);

// Subscribe — server stream as IAsyncEnumerable
var subs = client.Subscriptions("orders");
await foreach (var e in subs.SubscribeAsync(fromStart: true, ct))
{
    await Handle(e);
    await subs.AckAsync(e.Position, ct);
}
```

---

## 8. DI + hosting package (adoption lever + the Critter bridge)

`Reckon.Extensions.Hosting` — makes the SDK *feel* like Marten/Wolverine to a
.NET team, and is the concrete artifact behind the Codex appendix's "ReckonDB
event log + Marten read models" pattern:

```csharp
builder.Services.AddReckonClient(o => o.Address = "gateway:50051");  // singleton channel

// Base class for a subscription→projection worker (the Critter re-separation):
builder.Services.AddReckonSubscription<OrderProjection>(store: "orders");
```

- `AddReckonClient` — registers a singleton `ReckonClient`, options binding, a
  gRPC health check.
- `ReckonSubscriptionService<T>` — a `BackgroundService` base: subscribes over
  the SDK, dispatches each event to the handler, checkpoints position (in a
  reckon-db snapshot or the read DB). This is the worker that drives
  Marten-as-document-store from a reckon-db log. Ship it and the appendix stops
  being theory.

---

## 9. Milestones

| M | Deliverable |
|---|---|
| **M0** | Repo scaffold, proto submodule, Grpc.Tools build green, `Connect` + `Health.Check` round-trip vs lab gateway |
| **M1** | Streams: append/read/watch, `StreamState` sentinels, value records |
| **M2** | Subscriptions (`IAsyncEnumerable` + Ack + persistent create/checkpoint) + Snapshots |
| **M3** | DCB + CCC (`DcbFilter`, `DcbAppendResult`, payload reads on `DcbClient`) |
| **M4** | Schema, Temporal, Admin, Stores |
| **M5** | `Reckon.Extensions.Hosting`: `AddReckonClient` + `ReckonSubscriptionService<T>` + health check |
| **M6** | E2E suite vs beam cluster, README + examples, `dotnet pack` → NuGet packaging |

---

## 10. CI / release

- CI via the Codeberg→GitHub mirror + Actions (same as reckon-go): `dotnet build`
  + `dotnet test` + `dotnet pack`.
- E2E gated behind a lab gateway env var, like reckon-go's e2e.
- Version tracks the pinned reckon-proto tag (SemVer).
- **Package + docs prepared, then the human fires the NuGet publish.** Never
  auto-publish.

---

## 11. Open decisions (resolve before M0)

1. **Package name** — `Reckon.Client` (namespace `Reckon`, parity with
   reckon-go's `reckon`) vs `ReckonDb.Client` (avoids collision, more
   discoverable on NuGet). Leaning `Reckon.Client`.
2. **Proto sourcing** — git submodule (recommended, tightest to canonical) vs a
   `scripts/sync-proto.sh` vendoring copy (simpler for contributors who dislike
   submodules).
```
