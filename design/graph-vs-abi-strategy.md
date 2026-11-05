<!-- Extracted from Hermes forgebot skill: maven-compiler-plugin-incremental -->

# Graph vs ABI Strategy — Design Rationale

This document explains the two class-level incremental strategies (`graph` and `abi`),
their relationship, and the tradeoffs between them.

## Two orthogonal axes

The dependency graph and ABI fingerprinting are **separable** (they can be implemented
independently) but **combined in the ABI strategy** for maximum benefit.

### `graph` strategy

- Source → class → source tracking via classfile constant-pool scan.
- When a class changes (content hash), cascade fires to **all** sources that reference it.
- `abiFingerprint` is computed and stored but **NOT used for cascade decisions**.
- No cross-module manifest is written.
- Stores `GraphTypeInfo` entries in `IncrementalState`.

### `abi` strategy

- Same dep graph as `graph`, plus:
  - ABI fingerprint used to **prune cascade** — body-only changes don't cascade.
  - `abi-fingerprints` manifest written to `target/` after compilation.
  - `checkExternalAbiChanges()` reads upstream manifests for cross-module API change
    detection without recompiling upstream.
- Stores `AbiTypeInfo` entries in `IncrementalState`.

## The real distinction

The names `graph` and `abi` are somewhat misleading — both use `BytecodeAnalyzer` (classfile
API) and both analyze classfile deps internally. The real distinction is *scope and what ABI
data is used for*:

| Dimension | `graph` | `abi` |
|-----------|---------|-------|
| Classfile analysis | Constant-pool scan for intra-module deps | Same analysis, full sig/impl split |
| Dep storage | Union `classDeps` | Separate `signatureDeps` / `implementationDeps` |
| Cascade trigger | Any content-hash change | Only ABI fingerprint change |
| Cross-module | None (coarse JAR identity check) | Fine-grained type-level fingerprints |
| Manifest | Not written | `abi-fingerprints` written to `target/` |
| TypeInfo | `GraphTypeInfo` | `AbiTypeInfo` |

A naming like `graph` vs `graph-cross-module`, or `local` vs `reactor`, would better reflect
the actual distinction.

## Why `graph` without ABI is useful

Deps-only (`graph` without ABI) replaces the module-level rebuild with a class-level one,
which is strictly better for projects where most changes are isolated to leaf classes.

The `graph` + ABI combination is better still: private-body changes don't cascade at all.
But splitting deps from ABI into two implementation layers is a reasonable decomposition for
review and maintenance.

## Performance caveat for deps-only

For a widely-used type with N dependents, a private implementation change under deps-only
still cascades to all N dependents (because change detection is by content-hash, not API
surface). Each cascade fires a separate javac invocation rather than one efficient
full-module compile. For this specific scenario, deps-only is **slower** than
`sources,rebuild-on-change`.

ABI fingerprinting eliminates this regression by pruning body-only cascades.

## Performance comparison (Apache Maven reactor — 1,497 sources)

| Scenario | ABI strategy | sources+deps+rebuild-on-change |
|---|---|---|
| No-op | 49s | 48s |
| Wide API change (Session.java) | 1m03s | 1m21s |
| Isolated class change | ~48s | ~1m21s |
| Full clean build | 3m07s | 3m07s |

The gap grows with project size. `sources,rebuild-on-change` operates at **module
granularity**: any change triggers full recompile of the entire downstream module. ABI
operates at **class granularity** across modules.

## `implementationTypes` vs ABI fingerprint

- The `abiFingerprint` covers the public API surface only (`signatureTypes` — method/field
  signatures, supertypes, interfaces, exceptions, annotations).
- `implementationTypes` (method body instructions) are tracked for the dep graph but
  **excluded from the fingerprint**.
- A change to a method body:
  - Changes the content hash → cascade fires in `graph` (always)
  - Does NOT change the `abiFingerprint` → **no cascade** fires in `abi`
- This is the key precision win of the `abi` strategy.

## `ExternalAbiResolver` scope

- **Intra-module:** The dep graph (`signatureDeps`/`implementationDeps`) is used directly.
  `abiFingerprint` is NOT consulted.
- **Inter-module:** Fingerprints in the upstream manifest are compared to stored fingerprints
  in `IncrementalState.externalFingerprints`.

## `GraphIncrementalBuild.setAbiTracking(boolean)`

A single `abiTracking` flag controls the analysis branch:

- `false` (strategy `graph`): union of `signatureTypes`+`implementationTypes` as `classDeps`.
  Creates `GraphTypeInfo`. `finish()` does NOT write the manifest.
- `true` (strategy `abi`): full analysis. Creates `AbiTypeInfo`. `finish()` writes `AbiManifest`.

`ToolExecutor.compileWithGraphIncremental()` sets this flag based on the configured strategy.

## `incrementalStrategy` vs `incrementalCompilation` — design argument

### Why merging into one parameter is problematic

1. Semantics become heterogeneous — `"abi"` would be a mutually exclusive mode selector
   while `"sources"`, `"dependencies"` are additive aspect tokens.
2. The JDK < 24 fallback creates an awkward interaction: when `abi` falls back to timestamp,
   which aspect tokens apply?
3. `incrementalCompilation` cannot be used to combine algorithm selection with aspect tuning.

### Cleaner alternative

Rename `incrementalCompilation` to something like `timestampIncrementalAspects` to make it
explicit that it is timestamp-specific. `incrementalStrategy` stays as the orthogonal selector.
When `abi` falls back on JDK < 24, it naturally applies whatever `timestampIncrementalAspects`
is configured to.
