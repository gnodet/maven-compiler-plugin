<!-- Extracted from Hermes forgebot skill: maven-compiler-plugin-incremental -->

# IncrementalState Format

This document describes the binary `IncrementalState` format used to persist incremental
compilation state between builds.

## File location

The state file lives at:

```
target/maven-status/maven-compiler-plugin/<outputDirName>/incremental-state
```

No leading dot — files in `target/` are not meant to be hidden. Using `<outputDirName>`
(`classes` vs `test-classes`) avoids collisions between main and test compilation in the
same reactor module. Files inside `target/classes/` would be packaged into JARs by
`maven-jar-plugin` by default, so the state file is kept outside.

## Version scheme

Binary format uses a version int written first. The `load()` method gates new fields behind
`if (version >= N)`. Old state files with an older version return `null` → safe full rebuild.

```java
if (version < 1 || version > VERSION) {
    return null; // triggers full rebuild — safe fallback
}
```

### Version history

- **VERSION 1:** Source hashes + mtimes, type graph, classpath identities, configHash.
  Used by both dep-graph and ABI strategies during development.
- **VERSION 2:** Adds tag byte per `TypeInfo` entry for the sealed hierarchy
  (`0` = `GraphTypeInfo`, `1` = `AbiTypeInfo`). Migrates VERSION 1 on load.

### Versioning policy

Increment `VERSION` whenever the on-disk layout changes **at merge time** — do not
pre-increment for unreleased PRs. During development, stacked PRs keep the same version
until the format change ships to users and a state file from a prior release could exist
on disk.

## Content hash-based change detection

Correctness authority is the SHA-256 content hash, not mtime. Only an actual content change
causes recompilation — strictly better than pure mtime strategies for correctness across
checkouts and copies.

### Mtime-first optimization

`hashSourceFiles()` uses an **mtime-first short-circuit**: reads `BasicFileAttributes` and
compares the file's mtime against the stored value. If unchanged, the stored hash is reused
without reading file content. Only files whose mtime differs pay the SHA-256 cost. Mtimes
are persisted in `IncrementalState` (version 5+).

Consequence: `touch` alone does NOT trigger recompilation — the mtime changes but the
stored hash matches once the content is read.

## TypeInfo hierarchy

```java
public sealed interface TypeInfo permits GraphTypeInfo, AbiTypeInfo {
    String sourceFile();
    Set<String> classDeps();
    Set<String> annotationTypes();
    String moduleName();
}

public record GraphTypeInfo(
    String sourceFile,
    Set<String> classDeps,
    Set<String> annotationTypes,
    String moduleName
) implements TypeInfo {}

public record AbiTypeInfo(
    String sourceFile,
    Set<String> classDeps,
    Set<String> annotationTypes,
    String moduleName,
    Set<String> signatureDeps,
    Set<String> implementationDeps,
    String abiFingerprint
) implements TypeInfo {}
```

- **`GraphTypeInfo`:** All classfile constant-pool refs as `classDeps`. No fingerprint.
  All consumers indexed in a single `consumersIndex`.
- **`AbiTypeInfo`:** Full analysis — `signatureDeps` in `signatureConsumersIndex`,
  `implementationDeps` in `implementationConsumersIndex`, fingerprint for manifest.

### Serialization tag byte

`IncrementalState.save()`/`load()` write a tag byte before each entry: `0` = `GraphTypeInfo`,
`1` = `AbiTypeInfo`. VERSION 1 state is migrated on load by reconstructing `classDeps` as
the union of sig+impl.

## SourceFileAnalysis record

```java
public record SourceFileAnalysis(
    String qualifiedName,
    String sourceFile,
    Set<String> classDeps,          // always populated — constant pool union
    Set<String> signatureDeps,      // empty for graph strategy
    Set<String> implementationDeps, // empty for graph strategy
    String abiFingerprint,          // empty string for graph strategy
    String abiCanonical,            // empty string for graph strategy
    Set<String> annotationTypes,
    String moduleName
) {}
```

## Derived indexes (never serialized)

The following inverted indexes are always derived from the `types` map:

- `signatureConsumersIndex` — maps a type to all sources that have it in their `signatureDeps`
- `implementationConsumersIndex` — maps a type to all sources that have it in their `implementationDeps`
- `sourceToTypesIndex` — maps a source file to all types it produces

All three are rebuilt on `load()`, `copy()`, and `from()` and maintained incrementally by
`setType()` and `removeTypesForSource()`.

## Config hash

`computeConfigHash()` hashes both `module-info-patch.maven` file content AND
`configuration.options` (joined as a list). This ensures that changing `-source`, `-target`,
`-release`, `-parameters`, or custom `compilerArgs` between builds triggers a full rebuild.

## Classpath identities

For external change detection, the state stores classpath entry identities. The graph
strategy uses `size + ":" + lastModifiedTime().toMillis()` (not full content hash — hashing
entire JAR bytes would be a severe performance regression for large classpaths). For
directory entries, the `incremental-state` file's mtime is preferred over the directory's
own mtime.
