<!-- Extracted from Hermes forgebot skill: maven-compiler-plugin-incremental -->

# Incremental Compilation — Architecture Overview

This document describes the high-level architecture of the incremental compilation system
in the Maven Compiler Plugin, covering both the legacy timestamp-based approach and the
newer graph/ABI-based strategies.

## Evolution from 3.x to 4.x

### 3.x: `useIncrementalCompilation` (boolean)

In Maven Compiler Plugin 3.x, incremental compilation is controlled by a single boolean:

- **`true` (default):** Collects all source files upfront. Checks five triggers in order:
  immutable output, dependency changed (`DependencyState`), source mtime changed
  (`StaleSourceScanner`), input file tree changed (`hasInputFileTreeChanged` →
  `inputFiles.lst`), prior-execution overlap. Any trigger fires → **all sources passed
  to javac** (no per-file selection). Never delegates per-file decisions to javac.

- **`false` (not recommended):** `StaleSourceScanner` compares each `.java` mtime vs
  `.class` mtime. Only stale files passed to javac. No dependency checking at all.
  Risk: if `Foo.java` changes, `Bar.java` (which calls `Foo`) is NOT recompiled → stale `.class`.

### 4.x: `incrementalCompilation` (comma-separated Aspect values)

Parsed by `IncrementalBuild.Aspect.parse()`. Engine in `IncrementalBuild.java`,
orchestration in `ToolExecutor.java`.

**Aspects:**

| Aspect | Behavior |
|--------|----------|
| `options` | Full rebuild if compiler options hash changed |
| `dependencies` | Full rebuild if any classpath/modulepath file newer than previous build time |
| `sources` | Cache-based; removed files → delete stale `.class` + full rebuild; modified only → partial rebuild |
| `classes` | `.java` mtime vs `.class` mtime (equivalent to 3.x `false`) |
| `modules` | Delegates to `javac --module` — timestamp-based, not dependency-graph-based |
| `rebuild-on-add` | Modifier — any new file → full rebuild instead of partial |
| `rebuild-on-change` | Modifier — any modification → full rebuild instead of partial |
| `none` | Unconditionally pass all sources to javac |

**Default configurations:**

- **Base default (no AP, Java ≥ 23):** `options,dependencies,sources,rebuild-on-change` —
  full rebuild whenever any source file is modified. Safe: no stale `.class` from signature changes.
- **`proc=none` explicitly set:** `options,dependencies,sources` — NO `rebuild-on-change`.
  Per-file recompilation is safe because javac cannot discover or generate new source files.
- **Annotation processor present OR Java < 23:** additionally adds `rebuild-on-add` →
  full rebuild on any change.
- **Legacy `useIncrementalCompilation=true`:** maps to `dependencies,sources,rebuild-on-add`.
- **Legacy `useIncrementalCompilation=false`:** maps to `classes`.

### Why the mechanism changed

3.x `useIncrementalCompilation=true` always compiles **all** sources (no per-file selection).
It is correct but slow on large projects: even changing one file triggers full module recompilation.

4.x with the `sources` Aspect adds **true partial builds**: when only source files are modified
(no dependency change, no source added/removed), javac receives *only the changed files*
(`isPartialBuild=true`). The tradeoff: partial builds require the `outputDir` on the classpath
(so javac can resolve unchanged classes), which introduces the stale-class risk if the output
dir is also on the classpath during full rebuilds.

## Two incremental strategies

The plugin supports two orthogonal approaches for incremental compilation:

1. **Timestamp strategy** (default): Uses the Aspect system described above. Operates at
   module granularity — any dependency change triggers full recompile of the consuming module.

2. **Graph/ABI strategy** (opt-in): Uses class-level dependency tracking — a fundamentally
   different approach. Operates at **class granularity** across modules.

### `incrementalStrategy` vs `incrementalCompilation`

These two parameters are orthogonal:

- `incrementalCompilation` configures *which aspects* of the **timestamp strategy** trigger
  a rebuild: `sources`, `dependencies`, `rebuild-on-add`, `rebuild-on-change`, etc.
- `incrementalStrategy` selects the *algorithm*: `timestamp` (default), `graph`, or `abi`.

The `DEPENDENCIES` aspect in the timestamp strategy does NOT provide a dependency graph.
It detects that a classpath JAR's mtime changed and triggers a full module rebuild. These
are completely different things from the class-level dependency graph in the graph/ABI strategies.

## Post-compilation architecture

The graph/ABI strategy does **not** use `jdk.compiler` TaskListener. All dependency extraction
and ABI fingerprinting happens post-compilation from `.class` files. This makes it compatible
with `fork=true` and ECJ.

**Current class structure:**

- `ClassfileClassAnalyzer` (java24 source set) — extracts sig/impl split, annotation types,
  module name, `module-info.class` handling, constant pool scan; requires JDK 24+
- `BytecodeAnalyzer` — thin facade; hard-fails on JDK < 24 with clear message
- `BytecodeAnalyzer.ClassAnalysis` record — carries: `signatureTypes`, `implementationTypes`,
  `annotationTypes`, `moduleName`, `isModuleInfo`
- `GraphIncrementalBuild.processCompiledClasses(Set<Path>)` — replaces the old
  `attachTo(JavacTask)` + `processRound()` pair; called after each javac pass by `ToolExecutor`

### Recompilation loop

`processCompiledClasses` is called after each javac pass. Returns a non-empty set when cascade
is needed; `ToolExecutor` loops until fixpoint. Convergence guaranteed: `allCompiled` is
monotonically growing, bounded by total source count.

## Multi-release JAR (MRJAR)

All bytecode analysis uses `java.lang.classfile` (JDK 24+) exclusively — no ASM fallback for
incremental analysis (ASM remains for `ByteCodeTransformer`).

The plugin is a **proper multi-release JAR**:

| Location in JAR | Loaded on | Behaviour |
|---|---|---|
| `org/.../BytecodeAnalyzer.class` (root) | JDK < 24 | `isAvailable()=false`; throws `UnsupportedOperationException` |
| `META-INF/versions/24/org/.../BytecodeAnalyzer.class` | JDK 24+ | `isAvailable()=true`; delegates to `ClassfileClassAnalyzer` |
| `META-INF/versions/24/org/.../ClassfileClassAnalyzer.class` | JDK 24+ | Real implementation |

`AbstractCompilerMojo` checks `BytecodeAnalyzer.isAvailable()` before routing to the graph/ABI
strategy. If `false`, it logs a warning and falls back to the timestamp strategy — no exception,
no silent corruption.
