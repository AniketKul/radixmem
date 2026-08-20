# RadixMem

RadixMem is the working name for a standalone, coherent memory-control plane
for prefix KV caches in disaggregated LLM serving.

The central abstraction is a compressed global radix tree. Unlike a routing-only
prefix index, each tree node records physical KV-cache residency and participates
in the memory lifecycle. The intended control plane will provide atomic
match-and-lease, commit, release, migration, failure invalidation, and
subtree-aware eviction.

## Current status

The project is in architectural design. No implementation has started. The
selected first architecture is a standalone authoritative service with local or
mock memory workers; inference-engine adapters will follow after the core
semantics have been validated.

See [the current brainstorming record](docs/brainstorming/2026-08-20-current-status.md).

