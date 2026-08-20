# RadixMem

RadixMem is a standalone, coherent memory-control plane for prefix KV caches in
disaggregated LLM serving. It is inspired by MemServe's elastic memory pool and
global prompt trees, but focuses on making global cache metadata safe to act on.

## Problem

In a distributed serving system, KV-cache blocks may reside on different
workers and memory tiers such as GPU HBM, CPU DRAM, and SSD. A routing-only
global index can discover a useful prefix, but that prefix may be evicted,
migrated, or lost before the selected worker consumes it. The resulting stale
decision causes an unexpected miss or redundant prompt computation.

Local block-level eviction policies have a second limitation: they do not
understand prefix structure. Evicting a highly shared parent prefix can destroy
the value of an entire subtree even when its blocks appear individually cold.

## Approach

RadixMem uses a compressed global radix tree whose paths represent token
prefixes. Each cacheable node records:

- physical replicas and their worker and memory-tier locations;
- allocation size and lifecycle state;
- version and worker-health information;
- active leases; and
- access frequency and subtree utility.

The defining operation is **match-and-lease**:

```text
prompt -> longest usable cached prefix + protected replica lease
```

Matching and reservation happen atomically. While a lease is active, the
selected replica cannot be evicted. The client releases the lease after use,
and abandoned leases expire automatically.

The same tree coordinates:

- cache insertion and commit;
- longest-prefix lookup;
- replica placement;
- migration between memory tiers;
- lease-safe eviction;
- worker-failure invalidation; and
- subtree-aware retention decisions.

Unlike a best-effort routing directory, RadixMem is authoritative over the
cache-memory lifecycle.

## Objectives

RadixMem aims to:

- eliminate lookup-to-use eviction races;
- reduce unnecessary prompt recomputation;
- preserve high-value shared prefixes under memory pressure;
- make globally better eviction decisions than block-level LRU; and
- expose engine-neutral APIs suitable for future vLLM or SGLang integration.

## First milestone

The first implementation will run on a single host as a standalone service with
simulated workers and HBM, DRAM, and SSD tiers. It will implement the real radix
tree, concurrency control, lease state machine, placement logic, and eviction
policy without storing real KV tensors.

This setup lets us deterministically test concurrent operations, capacity
pressure, lease expiry, and worker failure before coupling correctness to a
particular inference engine or transport.

## Evaluation

We will compare RadixMem with:

- independent per-worker LRU;
- global block-level LRU; and
- a routing-only global radix index.

Primary measurements will include:

- recomputed prompt tokens;
- saved prefill work per cached byte;
- cache hit rate;
- eviction quality;
- lease contention and expiry behavior;
- control-plane latency; and
- correctness during worker failures.

## Initial non-goals

The first milestone will not include:

- LMCache or KVBM;
- real KV tensors or GPU-to-GPU transfers;
- vLLM or SGLang integration;
- a replicated or highly available RadixMem control plane; or
- a multi-node deployment.

## Current status

The project is in architectural design. No implementation has started. The
selected first architecture is a standalone authoritative service with local or
mock memory workers; inference-engine adapters will follow after the core
semantics have been validated.
