<!-- Extracted from Hermes forgebot skill: maven-compiler-plugin-incremental -->

# Stale Class Deletion

This document describes how stale `.class` files are detected and removed during
incremental compilation.

## 4.x: `inputFileTreeChanges`

In 4.x with the `sources` Aspect, `IncrementalBuild.inputFileTreeChanges()` handles removed
sources precisely:

1. Loads the source → output mapping from the previous-build cache.
2. Diffs against the current source file list.
3. For each source in the cache that no longer exists on disk → calls
   `SourceInfo.deleteClassFiles()`, which erases the primary `.class` file **and** all inner
   classes (`Foo$0.class`, `Foo$1.class`, …) by prefix-scanning the output directory.
4. Because a file was removed, returns a non-null `causeOfRebuild` → **full rebuild** of all
   remaining sources. `isPartialBuild` stays `false`.

### Cache dependency

`deleteClassFiles` only fires if the build cache from the previous run exists AND
`checkSources` is in `incrementalBuildConfig`. If the cache is absent (first build, after
`clean`, or cache corrupted), `inputFileTreeChanges()` is not called and stale `.class`
files from externally-deleted sources are NOT removed.

## Two sources of stale `.class` files

1. **Source file deleted** — `deleteClassFiles()` handles this correctly when the cache
   is present.
2. **Secondary class removed from an existing `.java` file** — e.g. a package-private
   `FooHelper` class removed from `Foo.java` that still exists. `inputFileTreeChanges()`
   does NOT detect this (the source file is still there); `deleteClassFiles()` does NOT
   delete `FooHelper.class` (it only knows the primary `.class` from the cache). The
   orphaned `.class` silently persists. This scenario requires output-ownership tracking
   to fix reliably and is a known pre-existing gap.

## 3.x stale class handling

In 3.x, the output directory is always on the classpath, unconditionally. Stale class
deletion is via `IncrementalBuildHelper.beforeRebuildExecution()` (deletes prior
`createdFiles.lst` outputs). If `BeanA.java` is deleted externally, 3.x has no mechanism
to detect and delete `BeanA.class` — the workaround is `mvn clean compile`.

## Partial build classpath — `setDependencyPaths()`

`setDependencyPaths()` in `ToolExecutor` adds the `outputDirectory` to the classpath
**only when `isPartialBuild=true`**:

```java
if (isPartialBuild && !hasModuleDeclaration) {
    paths = new ArrayDeque<>(paths);
    paths.add(outputDirectory);
}
```

- **Partial build:** javac receives only the modified files. It cannot resolve references
  to unchanged classes unless the output dir is on the classpath.
- **Full rebuild:** javac receives all sources. It resolves everything by compiling together —
  no need for the output dir, and adding it would expose stale `.class` files.

### ABI path exception

`compileWithAbiAnalyzer()` has a *separate, unconditional* block that adds `outputDirectory`
to the classpath regardless of `isPartialBuild`. This is intentional for the ABI strategy
(which always does per-file recompilation and needs previously-compiled classes visible).

### No-dependency edge case

When a project has zero classpath dependencies, the `dependencies` map contains no
`JavaPathType.CLASSES` entry. The loop body never executes the `CLASS_PATH` branch, so
`outputDirectory` is never added — even in `isPartialBuild` mode. The fix:

```java
if (isPartialBuild && !hasModuleDeclaration) {
    dependencies.putIfAbsent(JavaPathType.CLASSES, new ArrayDeque<>());
}
```

This must come *before* the classpath loop.

## Multi-release passes

For multi-release projects, the inter-pass classpath is handled by
`PathManager.configureSourcePaths()`, **independently of `isPartialBuild`**. After compiling
`src/main/java` → `target/classes`, the next pass (`src/main/java24`) gets
`latestOutputDirectory = target/classes` injected into its classpath. This is always-on
because the classes from the previous pass were just compiled in this build — they are
never stale.

This mechanism is **orthogonal** to the `isPartialBuild` guard in `setDependencyPaths()`.
