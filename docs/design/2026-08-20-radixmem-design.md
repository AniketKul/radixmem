# RadixMem Design Specification

- Status: Approved design, pending implementation plan
- Date: 2026-08-20
- Scope: Production-quality single-host control plane with simulated memory workers
- Language: C++20

## 1. Summary

RadixMem is an authoritative memory-control plane for reusable prefix KV cache.
It maintains a block-aligned compressed radix tree over exact token sequences and
coordinates the complete lifecycle of the physical cache objects represented by
that tree: publication, matching, leasing, placement, migration, eviction,
failure invalidation, and recovery.

The defining operation is `AcquirePrefix`, which performs longest-prefix matching
and lease acquisition as one externally atomic operation. A returned replica is
already protected against eviction. RadixMem therefore closes the race between a
routing or lookup decision and subsequent use of the selected KV memory.

The first release targets one GPU host with GPU HBM, CPU DRAM, and SSD tiers.
Memory workers are deterministic simulations behind the same interface intended
for later CUDA and NIXL adapters. The simulator is not permission to simplify the
control plane: concurrency, lifecycle state, capacity reservation, idempotency,
fencing, persistence, crash recovery, security isolation, and observability are
first-release requirements.

## 2. Motivation

A prompt index alone cannot guarantee safe cache reuse. Between matching a
prefix and consuming it, another operation can evict or migrate its cache,
invalidate its worker, or publish an incomplete replacement. Local LRU policies
also fail to model radix dependencies: removing a shared ancestor may eliminate
the value of a much larger hot subtree.

RadixMem treats the global prefix index as authoritative state rather than a
best-effort routing hint. It combines:

- exact, namespace-safe prefix identity;
- atomic match-and-lease;
- block and replica lifecycle state machines;
- tier-aware capacity reservations and data movement;
- subtree-aware admission and eviction;
- durable recovery and worker reconciliation; and
- deterministic simulation and adversarial correctness testing.

The design builds on the motivation of [MemServe](https://arxiv.org/abs/2406.17565),
the prefix structure of [SGLang RadixAttention](https://papers.nips.cc/paper_files/paper/2024/file/724be4472168f31ba1c9ac630f15dec8-Paper-Conference.pdf),
the block-level memory observations of [PagedAttention](https://arxiv.org/abs/2309.06180),
and the multi-tier transfer concerns demonstrated by [Mooncake](https://www.usenix.org/system/files/fast25-qin.pdf).

## 3. Goals

The first release must:

1. Implement a production-quality block-aligned compressed radix tree in C++20.
2. Guarantee that every returned prefix has compatible, ready, leased replicas.
3. Prevent lookup-to-use eviction and migration races.
4. Prevent partial or failed writes from becoming matchable.
5. Enforce exact byte accounting across committed, reserved, and in-flight data.
6. Make every externally mutating operation idempotent and fenced.
7. Recover safely from controller crashes, worker crashes, torn journal tails,
   interrupted operations, and full host reboots.
8. Isolate tenants and incompatible model or KV configurations.
9. Provide global LRU and independent cost-aware baselines plus a default
   subtree-aware, multi-tier policy.
10. Make correctness testable without a GPU through deterministic virtual time,
    failure injection, reference-model comparison, and replay.
11. Expose engine-neutral gRPC and C++ interfaces suitable for later vLLM,
    SGLang, CUDA, and NIXL adapters.

## 4. Non-goals for the first release

- Real KV tensors or attention-kernel integration
- Real CUDA, GPUDirect, RDMA, or NIXL transfers
- Multi-host deployment
- A replicated or highly available RadixMem controller
- Consensus across controllers
- Position-independent or non-prefix KV reuse
- KV compression or quantization algorithms
- Integration with LMCache or KVBM

Production quality in this specification refers to the standalone metadata and
lifecycle service. The complete GPU data path cannot be called production-ready
until later adapters pass hardware fault, load, and compatibility testing.

## 5. Core invariants

The following invariants are enforced in transaction validation and executable
debug/test assertions:

1. A match returns only a `READY` replica from the exact cache namespace.
2. Every returned replica has an active logical lease and confirmed physical pin.
3. A leased or transfer-pinned replica cannot enter physical reclamation.
4. A published prefix references a complete, ordered, compatible block manifest.
5. A match cannot cross a missing or non-ready ancestor.
6. Reserved plus committed bytes never exceed configured tier capacity.
7. Every allocation is owned by exactly one live replica or reservation.
8. Stale node, object, replica, lease, operation, controller, and worker
   generations can never become valid again.
9. Retrying an operation with the same idempotency key cannot duplicate effects.
10. Failed or cancelled transfers never publish their destinations.
11. Replicas from an old worker incarnation are never matchable.
12. Recovery and replay produce state equivalent to the committed history.
13. Cache state is never shared across security namespaces.
14. Capacity is not reported free until physical release is confirmed or the
    owning worker incarnation is declared lost.

An invariant failure quarantines the affected namespace and makes questionable
entries unavailable. It never converts uncertainty into a cache hit.

## 6. Architecture

```text
Client / engine adapter
          |
          v
 gRPC and C++ client API
          |
          v
     bounded shard router
          |
   +------+-----------------------+
   |                              |
   v                              v
state shard A                state shard B
radix index                  radix index
object catalog               object catalog
leases and pins              leases and pins
capacity accounting          capacity accounting
   |                              |
   +--------------+---------------+
                  |
                  +-- policy engine
                  +-- RocksDB durable store
                  +-- invariant checker
                  +-- metrics and tracing
                  +-- async worker executor
                              |
                              v
                    simulated worker adapter
                    CUDA/NIXL adapters later
```

### 6.1 Shard ownership

A stable hash of `CacheNamespaceId` assigns each namespace to one state shard.
Each shard has one owner thread and exclusively mutates its radix trees, object
catalog, leases, transaction records, and capacity views. RPC threads, timers,
durability callbacks, and worker callbacks submit commands through bounded
queues; none mutate state directly.

This produces a serial transaction order inside a namespace without a
fine-grained locking scheme. Independent namespaces execute concurrently.
Commands may be processed in bounded batches. Worker I/O and durable group
commit do not block the state-owner thread; their completions return as commands
that resume persisted transactions.

Each namespace has at most one durability-dependent mutation in flight. Later
commands for that namespace wait behind it, while the shard may continue serving
other namespaces and collecting additional transactions for group commit. This
preserves per-namespace transaction order without applying speculative state or
requiring rollback of visible mutations.

Cross-namespace transactions are not supported. Operations involving a hot
single namespace remain serialized in the first release and are protected with
admission control and explicit latency targets.

### 6.2 Separation of concerns

- `RadixIndex` owns logical prefix topology only.
- `ObjectCatalog` owns logical blocks and their physical replicas.
- `LeaseManager` owns logical leases and worker pin coordination.
- `CapacityManager` owns reservations and exact tier accounting.
- `TransactionEngine` validates and applies lifecycle transitions.
- `PolicyEngine` proposes legal actions from immutable state views.
- `WorkerRegistry` owns health, incarnation, and inventory knowledge.
- `DurableStore` owns transaction records, idempotency results, and snapshots.
- `WorkerAdapter` performs allocation, pinning, population, verification, copy,
  and release operations.

Policies and adapters cannot mutate live state. They return proposals or
completion commands that are revalidated by the owning transaction shard.

## 7. Cache identity and namespace isolation

A `CacheNamespace` contains compatibility and security identity:

- model weights fingerprint and revision;
- tokenizer fingerprint;
- adapter or LoRA identity and version;
- KV dtype and quantization parameters;
- attention, position, and RoPE configuration;
- KV physical-layout compatibility identifier;
- reusable block size;
- multimodal or prompt-embedding fingerprints where applicable; and
- authenticated tenant or trust-domain identity.

Exact token IDs are the authoritative radix key. Cryptographic hashes accelerate
lookup and identify external manifests, but exact stored tokens resolve every
collision. Tenant salts are derived by the service from authenticated identity;
a caller cannot opt into another tenant's cache.

Namespace configuration is immutable while objects exist. A changed model,
adapter, layout, or block size creates a new namespace rather than mutating the
meaning of existing entries.

## 8. Block-aligned compressed radix tree

A prompt is partitioned into complete blocks using the namespace's reuse
granularity. A block size of one provides token-granular behavior; larger sizes
match paged inference engines. An incomplete final block is request-private and
cannot be published for cross-request reuse.

The radix key is a sequence of exact `TokenBlock` values. Each compressed edge
stores one or more block units and their stable `CacheObjectId` values. Children
are bucketed by the next block's digest and verified against exact tokens.

Splitting or merging an edge changes metadata only. It does not move physical
cache bytes. Nodes are owned internally by the shard and referenced externally
only through generation-checked handles:

```cpp
struct NodeId {
  std::uint32_t slot;
  std::uint32_t generation;
};
```

Equivalent handles are used for objects, replicas, buffers, leases, workers, and
operations. Raw pointers never cross a shard, asynchronous boundary, or public
API.

A lookup walks to the deepest contiguous sequence of logical blocks for which a
compatible `READY` replica can be leased. A retained descendant behind a missing
ancestor is not returned. Metadata-only ghost nodes may remain for admission
statistics and are compacted only when they have no replicas, descendants,
claims, leases, or retained policy value.

## 9. Object and replica model

Each complete logical block has one `CacheObject` containing:

- stable ID and generation;
- namespace and exact token range;
- prefix depth;
- byte size and KV layout;
- manifest digest;
- access and subtree statistics; and
- zero or more physical replicas.

Each `Replica` contains:

- worker ID and worker incarnation;
- HBM, DRAM, or SSD tier;
- allocation handle and generation;
- lifecycle state;
- logical lease count;
- physical and transfer-pin counts;
- verification metadata; and
- timestamps and policy measurements.

Replica lifecycle:

```text
ABSENT
  -> RESERVED
  -> POPULATING
  -> VERIFYING
  -> READY
  -> DRAINING
  -> EVICTED
```

Failures before `READY` release or quarantine the reservation and never become
visible. `DRAINING` replicas are removed from new matches before reclamation and
remain allocated until all pins are gone and release is confirmed.

## 10. Public and worker APIs

### 10.1 Client API

- `AcquirePrefix`
- `RenewLease`
- `ReleaseLease`
- `BeginPopulation`
- `CommitPopulation`
- `AbortPopulation`
- `GetOperation`
- `GetServerInfo`
- `InspectPrefix`

Every mutating request includes a client identity, idempotency key, controller
epoch, deadline, and cache namespace. Reusing an idempotency key with different
parameters is rejected. A normal cache miss is a successful response with zero
matched blocks.

### 10.2 Worker API

- `ReserveAllocation`
- `ReleaseAllocation`
- `PinReplica`
- `UnpinReplica`
- `Populate`
- `Copy`
- `Verify`
- `CancelOperation`
- `GetOperationStatus`
- `ReportInventory`
- `Heartbeat`

Worker mutations carry the controller epoch, worker incarnation, object and
allocation generations, and operation ID. Workers reject stale commands.

## 11. Transaction flows

### 11.1 AcquirePrefix

```text
validate namespace and request
-> traverse radix tree
-> choose deepest contiguous reusable blocks
-> select compatible physical replicas
-> create PENDING lease and exclude replicas from eviction
-> persist transaction phase
-> establish worker pins
-> persist ACTIVE lease
-> return ordered manifest and lease token
```

If pinning one replica fails, RadixMem may select another. If no selected replica
can be pinned, the transaction rolls back and returns a miss or availability
status. A response never contains an unpinned allocation.

### 11.2 Population and stampede suppression

`BeginPopulation` creates a claim for a missing suffix and returns one of:

- `OWNER`: capacity is reserved for this client;
- `JOIN`: an equivalent population is already in progress; or
- `BYPASS`: inference may continue, but the result should not be cached.

The owner populates complete blocks. `CommitPopulation` verifies allocation,
manifest, generation, layout, size, and optional checksum before durably
publishing objects and modifying the tree. An abandoned claim expires and is
rolled back. If an equivalent object is published concurrently, policy either
keeps the allocation as a useful replica or releases it safely.

### 11.3 Lease renewal and release

Renewal must arrive before its cutoff, is bounded by configured maximum lifetime,
and cannot resurrect an expired lease. Release is idempotent. Lease expiration
removes worker pins but does not itself evict the object.

### 11.4 Eviction

```text
select legal candidate
-> mark DRAINING and remove from new matches
-> persist transition
-> wait for leases and transfer pins to reach zero
-> request physical release
-> confirm release or worker loss
-> update capacity and replica metadata
-> compact tree if legal
```

Removing one replica does not remove a logical object while another usable
replica exists.

### 11.5 Migration

```text
pin source
-> reserve and allocate destination
-> copy
-> verify destination
-> durably publish destination replica
-> release transfer pins
-> optionally drain source
```

Migration is never an in-place destructive move. A failed migration leaves its
source valid and cleans up only its incomplete destination. This structure is
compatible with later asynchronous registered-memory transports such as
[NIXL](https://github.com/ai-dynamo/nixl/blob/main/docs/doxygen/nixl_doxygen.md).

## 12. Capacity, admission, and policy

`CapacityManager` tracks committed, reserved, pinned, draining, and physically
free bytes separately for every tier. In-flight migration counts both source and
destination until source release is confirmed. No operation relies on an
optimistic free-space estimate.

Three replaceable eviction policies are included:

1. global LRU;
2. independent cost-aware eviction; and
3. default subtree-aware multi-tier eviction.

The default policy estimates the marginal loss of a legal action from expected
recomputation, transfer cost, descendant dependency loss, SLO impact, movement
cost, and bytes released. It uses decayed access statistics, matched depth,
subtree heat, object size, measured tier bandwidth and latency, replica count,
and tenant accounting group.

Legal actions include keeping, promoting, demoting, removing a redundant
replica, removing the final replica, and bypassing admission. The first release
does not create same-tier hot replicas on its one-GPU production target, although
the catalog supports multiple replicas for simulation and future expansion.

Admission uses a compact frequency sketch for rejected prefixes, saved-compute
per byte, existing claims, tier pressure, quotas, and estimated useful lifetime.
This prevents one-hit pollution while permitting first-use admission for large,
expensive prefixes.

High and low watermarks, minimum residency, promotion/demotion hysteresis,
movement cooldowns, migration concurrency limits, and emergency capacity reduce
thrashing. Under overload, population is rejected before lookup; releases,
failure processing, and transfer completion retain reserved queue capacity.

Every policy action records an explanation. The subtree-aware policy supports a
shadow mode in which a baseline makes the real decision while alternative
decisions are evaluated. Invalid or unavailable statistics fall back to
deterministic LRU among legal candidates. Policy can never bypass invariants.

## 13. Durability and crash recovery

RocksDB provides the first durable-store implementation. Application transactions
use versioned logical records in atomic `WriteBatch` operations with WAL enabled.
Acknowledged durability uses synchronous group commit. Periodic checksummed
snapshots bound replay time; a snapshot is installed only after all of its chunks
and terminal manifest are durable.

For a state transition, the shard validates and constructs a deterministic
transaction, durability commits it, and the shard applies the same transaction
in memory. A crash after durable commit but before application is resolved by
replay. A crash after application but before response is resolved by persisted
idempotency results.

Startup proceeds through:

```text
STARTING
-> load latest valid snapshot
-> replay committed records
-> durably advance controller epoch
-> reconcile workers and storage
-> SERVING or DEGRADED
```

Controller-process crash, worker-process crash, and host reboot have distinct
semantics:

- After controller crash, surviving workers report allocations and pins.
- After worker crash, a new incarnation fences every old volatile replica.
- After host reboot, HBM and DRAM are considered lost. Only verified SSD objects
  may be restored.

During reconciliation, matching is disabled for uncertain replicas. Records and
worker inventory are compared; missing replicas become `LOST`, unknown
allocations become `ORPHANED`, incomplete destinations are discarded, and
capacity is recomputed from accepted physical inventory. Namespaces may become
ready independently.

Logical leases are reinforced by expiring worker-side physical pins. Lease tokens
include controller epoch, lease ID, replica generation, worker incarnation, and
expiration. After restart, old clients cannot renew or create operations under an
old epoch, but old releases remain idempotently acceptable. Reported old pins are
protected until reconciled or their bounded recovery deadline expires.

The committed journal is never silently skipped. A corrupt committed record
causes safe degraded startup. A truncated uncommitted tail is ignored according
to RocksDB recovery semantics. Recovery produces misses rather than questionable
reuse.

## 14. Error and overload behavior

Representative API statuses are:

- `INVALID_ARGUMENT`: malformed or incompatible namespace or manifest;
- `FAILED_PRECONDITION`: stale epoch, generation, or lifecycle transition;
- `RESOURCE_EXHAUSTED`: no legal capacity or all candidates pinned;
- `ABORTED`: conflicting transaction;
- `UNAVAILABLE`: required worker unavailable or namespace recovering;
- `DEADLINE_EXCEEDED`: outcome may be queried or retried idempotently; and
- `DATA_LOSS`: verification or committed-state corruption.

Queues are bounded. Cancellation is a lifecycle event, not thread interruption.
If an RPC times out after commit, `GetOperation` or an idempotent retry returns the
committed result. The service never waits indefinitely for evictable memory.

Shard command priority is approximately:

1. failure detection and fencing;
2. lease release and expiration;
3. transfer completion and rollback;
4. lease acquisition and renewal;
5. population commit;
6. new reservation; and
7. policy scans, snapshots, and compaction.

## 15. Security

- Unix-domain sockets are the default local endpoint.
- TCP is disabled unless configured and requires mutual TLS in production mode.
- Authenticated principals map to permitted namespaces and server-derived salts.
- Token count, block count, RPC size, lease count, and population concurrency are
  bounded per principal and globally.
- Logs, traces, and metric labels never contain prompt token values.
- Administrative inspection returns IDs, hashes, lengths, and states by default.
- Persistence contains sensitive token IDs and requires a protected, encrypted
  filesystem with restrictive ownership and permissions.
- Development-only authentication bypass requires an explicit flag and warning.

Cross-tenant cache-hit timing is a security concern, not only a performance
detail; this threat has been demonstrated by research on
[prompt leakage through KV-cache sharing](https://www.ndss-symposium.org/ndss-paper/i-know-what-you-asked-prompt-leakage-via-kv-cache-sharing-in-multi-tenant-llm-serving/).

## 16. Observability and operations

RadixMem exports structured logs, Prometheus-compatible metrics, and OpenTelemetry
traces. Telemetry covers RPC and queue latency, match depth, hits by tier,
capacity states, lease age and expiration, population joins and bypasses,
migration and rollback, reconciliation, policy decisions, durability latency,
recovery progress, and invariant checks.

Health is separated from readiness. During recovery, the process can be healthy
while one or more namespaces remain unready or degraded. A periodic and on-demand
deep invariant checker validates tree closure, handle generations, object and
replica ownership, capacity equations, leases, pins, and durable sequence
progress.

Configuration uses a validated TOML file with command-line overrides. Unsafe or
internally inconsistent capacity, timeout, security, and namespace settings fail
startup rather than being repaired implicitly.

## 17. C++ implementation constraints

- C++20 with a pinned Clang 18-or-newer primary toolchain
- CMake, CMake Presets, Ninja, and a vcpkg manifest
- gRPC callback API and Protobuf
- RocksDB
- GoogleTest and GoogleMock
- Google Benchmark
- Prometheus and OpenTelemetry C++ libraries
- Clang AddressSanitizer, UndefinedBehaviorSanitizer, and ThreadSanitizer
- libFuzzer and `clang-tidy`

Internal ownership uses RAII and single-shard ownership. No asynchronous callback
captures internal node or replica pointers. Size and offset calculations use
checked arithmetic. Exceptions do not cross RPC, persistence, worker, or state
machine boundaries. All dependencies and serialization formats are versioned and
pinned.

Suggested source boundaries:

```text
proto/                 public and worker protocols
include/radixmem/      stable public C++ interfaces
src/core/              IDs, models, radix index, state transitions
src/policy/            LRU, cost-aware, and subtree-aware policies
src/persistence/       RocksDB journal, snapshot, and replay
src/server/            gRPC, shard router, security, configuration
src/worker/            adapter contract and deterministic simulator
src/observability/     metrics, tracing, invariant reporting
tests/unit/            focused unit tests
tests/model/           reference-model and generated histories
tests/integration/     RPC, durability, and recovery tests
tests/fuzz/            libFuzzer targets
tests/chaos/           process crash and fault-injection scenarios
benchmarks/            core, service, recovery, and policy benchmarks
```

## 18. Verification strategy

A simple reference model stores complete block paths in standard containers.
Generated histories execute against both the reference and optimized
implementations and compare every observable result.

Testing includes:

- insertion splits at every legal edge boundary;
- prefixes of prefixes and duplicate publication;
- maximum-size and empty requests;
- lease acquisition, renewal, release, and expiration;
- capacity exhaustion and no-eviction states;
- ancestor and descendant eviction;
- worker loss and incarnation changes;
- migration, cancellation, verification failure, and rollback;
- idempotent retries before and after commit;
- generation rollover and stale-handle attacks;
- namespace and security isolation;
- deterministic virtual-clock behavior;
- failpoints before and after every durable phase;
- child-process termination, restart, and reconciliation;
- truncated uncommitted journal tails and corrupt committed records;
- recorded concurrent histories checked against shard commit order;
- queue saturation and slow worker callbacks; and
- fuzzing of Protobuf decoding, radix operations, journal recovery, and
  lifecycle transitions.

CI treats warnings as errors and runs unit, model, integration, static-analysis,
sanitizer, fuzz-smoke, and benchmark-regression configurations. ThreadSanitizer
runs separately from other sanitizers. A longer scheduled job performs crash
matrices and soak testing.

## 19. First-release acceptance criteria

Correctness gates:

- ten million deterministic randomized state-machine operations without
  reference-model divergence;
- every lifecycle failpoint exercised through crash and recovery;
- no sanitizer findings;
- no invariant violations in a 24-hour high-contention soak test;
- safe recovery from truncated uncommitted tails and fail-closed startup for
  corrupt committed records; and
- complete cross-namespace isolation tests.

Initial single-host performance goals:

- one million logical cache blocks;
- in-process radix match p99 below 100 microseconds;
- local `AcquirePrefix` p99 below 2 milliseconds at 5,000 operations per second,
  including durable lease establishment;
- recovery of one million objects within 60 seconds;
- no unbounded queues or memory growth;
- metadata memory below 512 bytes per logical block, excluding configured
  RocksDB caches; and
- background policy and snapshots add less than 5% p99 latency overhead.

Policy evaluation compares per-worker LRU, global LRU, independent cost-aware
eviction, and subtree-aware multi-tier eviction. Metrics include recomputed
tokens, saved prefill time, transfer bytes, hit rate, tier occupancy, eviction
quality, and tail latency. The target is at least a 10% reduction in recomputed
prompt tokens versus global LRU on prefix-sharing workloads without more than a
5% control-plane p99 regression. Neutral and adverse workloads are reported.

Performance numbers are engineering acceptance goals, not assumed claims. A
missed target requires a reproducible benchmark, profile, and documented
bottleneck before release disposition.

## 20. Principal risks and mitigations

| Risk | Mitigation |
| --- | --- |
| Hot namespace serializes all mutations | Bounded batching, low-allocation core, overload control, benchmark gate |
| Durable leases increase p99 latency | Worker pins, synchronous group commit, measured batching window |
| RocksDB state diverges from memory | Deterministic transaction records, sequence checks, replay tests |
| Subtree policy is expensive | Immutable snapshots, bounded candidate sets, shadow mode, LRU fallback |
| Tree metadata grows after physical eviction | Ghost-node budget and safe compaction |
| Lease abuse pins all capacity | Quotas, maximum lifetime, renewal cutoff, backpressure |
| Worker callback arrives after reuse | Operation and allocation generations plus controller and worker fencing |
| Simulator hides hardware assumptions | Narrow adapter contract, explicit layout identity, later conformance suite |
| Persisted tokens expose prompts | Namespace authorization, no token telemetry, encrypted protected storage |

## 21. Approved decisions

- C++20 is the end-to-end implementation language.
- RadixMem is an authoritative standalone service, not a routing-only index.
- The first release targets a single GPU host.
- The radix tree is compressed and block-aligned over exact token blocks.
- Match and lease are one externally atomic operation.
- Physical objects and replicas are separate from radix topology.
- Namespace-sharded single-writer transaction engines are used instead of a
  lock-free or fine-grained-locking tree.
- Generation-checked IDs cross asynchronous boundaries; raw pointers do not.
- Worker-side pins reinforce logical leases.
- Publication, migration, eviction, and recovery use explicit state machines.
- RocksDB-backed journal, snapshots, and reconciliation are first-release scope.
- Full host reboot loses HBM and DRAM; only verified SSD replicas survive.
- Subtree-aware multi-tier policy is the default; LRU and independent cost-aware
  policies are baselines.
- Security, observability, fault injection, and acceptance gates are part of the
  first implementation rather than follow-up hardening.
