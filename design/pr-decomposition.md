<!-- Extracted from Hermes forgebot skill: maven-compiler-plugin-incremental -->

# PR Decomposition — Incremental Compilation Feature

This document describes the decomposition of the incremental compilation work into separate
PRs for review and the rationale behind the layering.

## Conceptual decomposition (4 steps)

External reviewer M. Desruisseaux proposed splitting the ABI work into 4 independent steps:

1. **Dependency graph** — replace one-to-one source ↔ class with one-to-many (class-level
   cascade on change). → PR #1157 (`graph` strategy)
2. **ABI filtering** — replace "modified files" with "effectively modified from API consumer
   POV" via fingerprinting (body-only changes don't cascade). → PR #1155 (within-module part)
3. **ABI manifest** — persist fingerprints for cross-module/sub-project change detection
   without re-reading bytecode. → PR #1155 (`AbiManifest` + `ExternalAbiResolver`). Bundled
   with step 2 because `AbiManifest.write()` lives in `finish()` alongside the fingerprint
   computation.
4. **Processor classification** — use `META-INF` metadata to classify annotation processors
   as isolating/aggregating/unknown. → PR #1157 (`ProcessorClassification`) + PR #1161.

## PR structure

### PR A: `feature/mrjar-bytecode-analyzer` → master

MRJAR infrastructure + `ByteCodeTransformer` only.

**Files:** `BytecodeAnalyzer.java` (root + java24), `ClassfileClassAnalyzer.java`,
`Sha256.java`, `BytecodeAnalyzerTest.java`.

`AbstractCompilerMojo` and `ToolExecutor` are **unchanged from master**.

### PR B: `feature/dep-graph-incremental` → master

Dep-graph strategy (`incrementalStrategy=graph`). Engine class: `GraphIncrementalBuild`.

Key design decisions:
- Strategy name: `"graph"` (not `"abi"`)
- State file: `incremental-state` (no dot, no hidden file convention)
- `IncrementalState.TypeInfo`: simple record — no sealed interface, no ABI fields
- Consumer lookup via single `consumersIndex`
- Cascade on ANY compiled type (content-hash change)
- External change detection via JAR identity comparison (not type-level fingerprints)
- No `AbiManifest`, no `ExternalAbiResolver`

### PR C: `feature/abi-strategy` → PR B → master

ABI fingerprinting on top of PR B.

Additions:
- `AbiTypeInfo` to the `TypeInfo` sealed hierarchy
- `GraphIncrementalBuild.setAbiTracking(boolean)` flag
- `AbiManifest`, `ExternalAbiResolver`, cross-module fingerprint exchange
- `IncrementalState` VERSION bump (adds tag byte per entry)
- Strategy name: `"abi"` (routes to same `compileWithGraphIncremental`; `setAbiTracking`
  distinguishes internally)

## Why steps are bundled

**Steps 1+4 (graph + processor classification):** Classification already has behavioral
impact at the graph level — aggregating processor → force full rebuild regardless of cascade.
Not independent of step 1.

**Steps 2+3 (ABI filtering + manifest):** The coupling is in `finish()` — `AbiManifest`
writes exactly the fingerprints that the ABI comparison computed. A theoretical split would
produce a thin second PR.

## Branch restructuring procedure

When splitting a mono-branch into two clean PRs:

1. Create graph-only branch from master: `git checkout -B new-pr-B origin/master`
2. Checkout ALL changed files from the mono-branch
3. Remove ABI-only files via `git rm`
4. Edit shared files to strip ABI — file by file
5. Run tests — must pass before commit
6. Commit + push
7. Create ABI branch from graph-only: `git checkout -B new-pr-C new-pr-B`
8. Restore ABI files from mono-branch
9. Restore full versions of shared files from mono-branch
10. Run tests — must pass before commit
11. Commit + push

Key: step 7–11 uses `git checkout` from the mono-branch on top of the graph-only branch —
no cherry-pick, no patch application. The diff between PR B and PR C is exactly the ABI
layer addition.
