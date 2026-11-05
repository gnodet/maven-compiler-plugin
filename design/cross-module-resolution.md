<!-- Extracted from Hermes forgebot skill: maven-compiler-plugin-incremental -->

# Cross-Module ABI Resolution

This document describes how the ABI strategy detects and propagates API changes across
Maven module boundaries.

## Two-file architecture

### IncrementalState (private, binary)

- Location: `target/maven-status/maven-compiler-plugin/<outputDirName>/incremental-state`
- Private incremental bookkeeping for the current module
- Binary, versioned format
- NOT meant for external readers

### ABI manifest (public, text)

- Location: `target/abi-fingerprints`
- **Public API boundary** between modules
- Text format: `type=fingerprint` (one per line)
- Intentionally readable and forward-compatible across plugin versions
- Minimal — type → fingerprint only

These are deliberately separate. Merging them would conflate private bookkeeping with the
public inter-module interface. Downstream modules should never need to load the full
upstream `IncrementalState`.

## ExternalAbiResolver

Handles inter-module deps only. For intra-module, the dep graph
(`signatureDeps`/`implementationDeps`) is used directly.

### Resolution strategies

`ExternalAbiResolver.resolveFromManifests()` reads upstream `abi-fingerprints` manifests
from reactor sibling modules. Three strategies, in order:

1. **Manifest** — Read `target/abi-fingerprints` from directory classpath entries.
   Fastest — no bytecode scanning.
2. **Reactor bytecode** — For reactor modules without a manifest. Analyze `.class` files
   directly.
3. **JAR fallback** — For non-reactor JARs. Scan JAR entries for external type fingerprints.
   Most expensive on large classpaths.

### Maven 4 project-local-repo issue

Under Maven 4, reactor sibling JARs may come from
`[root]/.mvn/target/project-local-repo/…/artifact.jar`. These fail `Files.isDirectory()`,
skip the manifest strategy entirely, and fall through to bytecode analysis.

Fix: build a `Map<Path jarPath, Path targetDir>` from `session.getProjects()` and pass it
to `ExternalAbiResolver` so JAR entries that are reactor artifacts can still look up their
manifest.

## `checkExternalAbiChanges()` ordering

This method **must be called BEFORE the no-local-changes early return** in
`initIncrementalBuild()`. If placed after the guard that returns `Set.of()` when
`changedFiles`, `newFiles`, and `deletedFiles` are all empty, a module with zero local
changes (the normal case when only an upstream ABI changed) exits without ever consulting
the manifest.

Correct structure:
1. Compute local change sets
2. Call `checkExternalAbiChanges()`
3. Return early only if ALL categories (local + external) are empty

## ABI symmetry

The ABI approach is symmetric across intra- and inter-module tracking:

- **Intra-module:** `signatureTypes` in the dep graph
- **Inter-module:** `abi-fingerprints` manifest

Both express "recompile this only if the public API changed." The `abiFingerprint` in
`TypeInfo` is the per-type hash of `signatureTypes`; the manifest aggregates these hashes
for upstream → downstream communication.

A change to `implementationTypes` only:
- Changes content hash in `graph` (cascade fires)
- Does NOT change `abiFingerprint` (no cascade in `abi`, neither intra- nor inter-module)

## Graph strategy cross-module (coarse-grained)

Without ABI fingerprints, the `graph` strategy uses a simpler approach:

- `computeCurrentClasspathIdentities()` computes `Map<String, String>` of classpath entry →
  identity (file size + mtime, not full content hash).
- Any changed or disappeared JAR triggers **broad invalidation**: recompile ALL sources.
- This coarser-grained detection is acceptable because graph-level cascade already recompiles
  all consumers of a changed class anyway.
