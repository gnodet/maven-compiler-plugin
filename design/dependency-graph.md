<!-- Extracted from Hermes forgebot skill: maven-compiler-plugin-incremental -->

# Class Dependency Graph

This document describes the class-level dependency graph used by both the `graph` and `abi`
incremental strategies.

## Source → class → source tracking

The dependency graph provides source → class → source tracking. When a class changes, only
sources that actually use it are recompiled. This is fundamentally different from the
timestamp strategy's module-level `DEPENDENCIES` aspect, which merely detects that a
classpath JAR's mtime changed and triggers a full module rebuild.

## Sig/impl split from bytecode

`ClassfileClassAnalyzer` extracts two categories of type references:

| Source | Category |
|--------|----------|
| Non-private field/method descriptors | `signatureTypes` |
| Private member descriptors | `implementationTypes` |
| Bytecode instructions (INVOKE*, NEW, CHECKCAST, field owners) | `implementationTypes` |
| Constant pool class entries | `implementationTypes` |
| Annotation types | `annotationTypes` (separate) |

### Constant pool scan — inlined constants

When class `Service` references `Constants.MAX` (a `static final int`), javac inlines the
value and emits NO GETSTATIC instruction in `Service.class`. The reference to `api.Constants`
survives only as a `ClassEntry` in the constant pool. Instruction-only scanning would miss
this dependency entirely. Fix: after scanning instructions, iterate `cm.constantPool()` for
`ClassEntry` instances and add non-array, non-self entries to `implementationTypes`.

## Source → class mapping strategy

`collectClassAnalyses(sourceFile, results)` maps a source file to its produced class files via
the **`SourceFile` class file attribute** — not simple class name. This correctly handles:

- Package-private secondary top-level types (`FooHelper` in `Foo.java`)
- Cross-package simple-name collisions (`api.Model` vs `other.Model`)

`BytecodeAnalyzer.ClassAnalysis` carries `sourceFileName` for this purpose.

### Resolution strategies (in priority order)

1. **Previous state** (incremental, known types): look up `previousState.getTypesFromSource()`
   → resolve each to a `.class` path, also scan for inner classes (`Foo$Bar.class`).
2. **Package dir scan** (incremental, new types in known package): `inferPackageDir()`
   returns the output package directory from previous state; scan for `SimpleName.class` and
   `SimpleName$*.class`.
3. **Full output dir scan** (full build OR new file with no previous state): walk the entire
   output directory once, match class files by `SourceFile` attribute. Result cached per
   `processCompiledClasses` call.
4. **module-info special case**: always add `outputDir/module-info.class` if it exists and
   source is `module-info.java`.

### Double-analysis elimination

`collectClassAnalyses` uses a `LinkedHashMap<Path, BytecodeAnalyzer.ClassAnalysis>` cache.
The package-dir walk stores the resulting `ClassAnalysis` as the map value. The final
analysis pass skips `BytecodeAnalyzer.analyze()` for entries that already have a non-null
analysis.

### Known gap

Package-private secondary classes in the same file (e.g. `Foo.java` contains
`class FooHelper {}`) are NOT matched by the simple-name strategy — only `Foo.class` and
`Foo$*.class` are found. `FooHelper.class` persists correctly but is not attributed to
the source, so its dependency edges are invisible until the next full build.

## Recompilation loop

`processCompiledClasses` is called after each javac pass. Returns a non-empty set when
cascade is needed; `ToolExecutor` loops until fixpoint.

**Convergence guarantee:** `allCompiled` is monotonically growing, bounded by total source
count. Handles new inter-file dependencies introduced simultaneously with content changes
in both files.

## Consumer indexes

The dependency graph maintains inverted indexes for efficient cascade lookup:

- **`graph` strategy:** single `consumersIndex` — all classfile constant-pool refs indexed together
- **`abi` strategy:** dual indexes — `signatureConsumersIndex` (API-surface deps) and
  `implementationConsumersIndex` (body-only deps)

All indexes are rebuilt on `load()`/`copy()`/`from()` and maintained incrementally by
`setType()` and `removeTypesForSource()`. They are **never serialized** — always derived
from the `types` map.

## Cross-module dependency tracking

### Graph strategy (coarse-grained)

`checkExternalAbiChanges()` computes a `Map<String, String>` of classpath entry → identity
(file size + mtime for JARs, `incremental-state` file mtime for directories). Any changed
or disappeared entry triggers **broad invalidation: recompile ALL sources**. No per-type
resolution is possible without ABI fingerprints.

### ABI strategy (fine-grained)

The `ExternalAbiResolver` reads upstream `abi-fingerprints` manifests from reactor sibling
modules. Changed type fingerprints are compared to stored fingerprints in
`IncrementalState.externalFingerprints`, allowing per-type cascade across module boundaries.

The ABI approach is symmetric across intra- and inter-module tracking: `signatureTypes`
(intra-module ABI dep graph) and the cross-module `abi-fingerprints` manifest serve the
same purpose at different scopes.
