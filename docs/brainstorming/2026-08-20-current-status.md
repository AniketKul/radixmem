# RadixMem brainstorming status

Date: 2026-08-20

## Goal

Build a useful memory-management component inspired by MemServe's elastic
memory pool and global prompt trees, while addressing a narrower correctness
gap: a global prefix match must remain usable between scheduling and KV-cache
consumption.

## Selected direction

Use a standalone authoritative control-plane service backed by a compressed
global radix tree.

Each prefix node will eventually describe:

- the model and token-prefix identity;
- one or more physical KV-cache replicas;
- worker and memory-tier location;
- lifecycle state and version;
- active leases that temporarily prevent eviction;
- size, access, and subtree-utility statistics.

The defining operation is `match-and-lease`: return the longest usable prefix
and atomically protect its selected replica for a bounded interval. This closes
the race in which routing observes a cached prefix immediately before a worker
evicts or migrates it.

## First milestone boundary

- Standalone service, not embedded in an inference scheduler.
- Local/mock workers simulating HBM, DRAM, and SSD capacities.
- Real radix-tree indexing, leases, placement metadata, and eviction decisions.
- Deterministic concurrency and failure tests on a single host.
- No vLLM or SGLang integration in the first milestone.
- No replicated control plane in the first milestone.
- No real KV tensors or GPU data transfer in the first milestone.

## Why this is more than a global prefix index

MemServe and Preble already use global prompt trees for locality-aware
scheduling. FlexKV and contemporary serving stacks also maintain distributed
prefix-locality indexes. A routing-only index therefore has limited novelty.
RadixMem instead makes the tree authoritative over memory state and exposes a
transaction-like lifecycle between lookup, reservation, materialization, and
release.

## Decisions made

1. Do not use LMCache or KVBM as the cache manager.
2. Start with a new standalone repository.
3. Target a design that is testable on one H100 host.
4. Select the coherent centralized control-plane architecture over replicated
   tree snapshots or a scheduler-embedded routing index.
5. Keep future inference-engine integration behind worker and transport APIs.

## Design work remaining

1. Define the exact worker and tier model.
2. Define radix node identity and compressed-edge representation.
3. Specify lifecycle and lease state machines.
4. Specify public APIs and consistency guarantees.
5. Choose placement and subtree-aware eviction objectives.
6. Define crash, timeout, and stale-worker behavior.
7. Define simulator workloads, invariants, and success metrics.
8. Present the complete design for review before implementation.

## Background reading

- [MemServe: Flexible Mem Pool for Building Disaggregated LLM Serving with Caching](https://arxiv.org/abs/2406.17565)
- [Preble: Efficient Distributed Prompt Scheduling for LLM Serving](https://arxiv.org/abs/2407.00023)
- [SGLang: Efficient Execution of Structured Language Model Programs](https://www.usenix.org/conference/osdi24/presentation/zheng-lianmin)
- [Mooncake: Trading More Storage for Less Computation](https://www.usenix.org/conference/fast25/presentation/qin)
- [KVCache Cache in the Wild](https://www.usenix.org/conference/atc25/presentation/wang-jiahao)

## Preserved visual artifacts

The earlier architecture-comparison HTML and click-event records were moved
under `.superpowers/brainstorm/` in this repository. They are intentionally
ignored by Git because the directory also contains transient browser-server
state.
