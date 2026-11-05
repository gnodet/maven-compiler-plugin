<!-- Extracted from Hermes forgebot skill: maven-compiler-plugin-incremental -->

# Incremental Compilation — Design Documentation

Design documents for the Maven Compiler Plugin's incremental compilation system, covering
both the timestamp-based Aspect strategy and the graph/ABI class-level strategies.

## Documents

| File | Description |
|------|-------------|
| [incremental-compilation-overview.md](incremental-compilation-overview.md) | High-level architecture: 3.x → 4.x evolution, Aspect system, strategy overview, MRJAR design |
| [abi-fingerprint.md](abi-fingerprint.md) | ABI fingerprinting pipeline: canonical form, cost profile, cascade precision, design decisions |
| [dependency-graph.md](dependency-graph.md) | Class dependency graph: sig/impl split, source → class mapping, recompilation loop, consumer indexes |
| [incremental-state.md](incremental-state.md) | IncrementalState binary format: versioning, TypeInfo hierarchy, SourceFileAnalysis, derived indexes |
| [graph-vs-abi-strategy.md](graph-vs-abi-strategy.md) | Design rationale: graph vs ABI strategy comparison, tradeoffs, performance data |
| [type-system-tracking.md](type-system-tracking.md) | Generic type tracking, JVM signature parsing, bridge methods, annotation processor classification |
| [stale-class-deletion.md](stale-class-deletion.md) | Stale `.class` detection and removal, partial build classpath, multi-release pass handling |
| [cross-module-resolution.md](cross-module-resolution.md) | Cross-module ABI resolution: manifest architecture, ExternalAbiResolver, Maven 4 local-repo |
| [pr-decomposition.md](pr-decomposition.md) | PR decomposition: 4-step conceptual split, layer assignments, branch restructuring procedure |

## Reading order

For newcomers to the codebase, suggested reading order:

1. **incremental-compilation-overview.md** — start here for the big picture
2. **graph-vs-abi-strategy.md** — understand the two strategies and why both exist
3. **dependency-graph.md** — how class-level deps are extracted and tracked
4. **abi-fingerprint.md** — how ABI fingerprints work and their precision advantage
5. **type-system-tracking.md** — details of generic type and annotation tracking
6. **incremental-state.md** — the on-disk format
7. **cross-module-resolution.md** — how changes propagate across Maven modules
8. **stale-class-deletion.md** — edge cases in class file lifecycle
9. **pr-decomposition.md** — how the implementation is structured for review
