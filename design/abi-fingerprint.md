<!-- Extracted from Hermes forgebot skill: maven-compiler-plugin-incremental -->

# ABI Fingerprint Design

This document describes the ABI (Application Binary Interface) fingerprinting system used
by the `abi` incremental strategy to determine whether a class's public API has changed.

## Pipeline overview

```
classfile bytes → buildCanonicalForm() → String (transient) → Sha256.hash() → 16-char hex fingerprint
```

The fingerprint is stored in `IncrementalState` (per-type `AbiTypeInfo`) and in the
cross-module `abi-fingerprints` manifest. The canonical string (`abiCanonical`) is **never
persisted** — it is transient in `SourceFileAnalysis` and `BytecodeAnalyzer.ClassAnalysis`,
both discarded at the end of `processCompiledClasses()`.

## Cost profile

The string canonical form is not the bottleneck. Cost hierarchy per class file:

1. `BytecodeAnalyzer.analyze()` — classfile I/O + constant pool walk (**dominates**)
2. `buildCanonicalForm()` — StringBuilder appends over a handful of fields/methods (negligible)
3. `Sha256.hash(string)` — SHA-256 on ~hundreds of bytes of canonical text (negligible)
4. `String.equals()` on two 16-char fingerprints (negligible)

A binary canonical form would save only step 2 (UTF-8 encoding overhead). The gain is
immeasurable against step 1. The readability of a human-readable canonical and manifest
outweighs zero measurable speedup.

## `buildCanonicalForm()` coverage

The canonical form covers the public API surface only:

- Class access flags (public/protected/abstract/static/final, interface/enum/class keyword)
- Generic class signature (Signature attribute)
- Superclass (excluding `java.lang.Object`, `java.lang.Enum`, `java.lang.Record`)
- Implemented interfaces (**sorted** — declaration order is not semantically significant)
- Non-private, non-synthetic fields: access flags + JVM descriptor + name + constant value + generic signature
- Non-private, non-synthetic methods: access flags + name + descriptor (params + return) + generic signature

**Excluded** (intentionally, body-only): method bytecode, private members, synthetic bridge
methods, local variable tables, line numbers.

### Interface sort requirement

`cm.interfaces()` returns declaration order. Reordering `implements A, B` to
`implements B, A` in source would produce a different ABI fingerprint even though it is
semantically equivalent. The interface stream must be `.sorted()` before collecting to the
canonical form.

### Type parameter bounds

`buildCanonical()` must sort intersection type bounds alphabetically before joining with
` & `. The Classfile API does not guarantee order; non-deterministic fingerprints would cause
spurious cascade on every build.

## ABI cascade — body-only vs signature change

When a type is recompiled:

1. Compare previous `abiFingerprint` vs new `abiFingerprint` (16-char string `equals()`).
2. **Fingerprint unchanged** → body-only change → cascade only to
   `implementationConsumersIndex` entries, not `signatureConsumersIndex`.
3. **Fingerprint changed** → API surface changed → cascade to all consumers via
   `expandSignatureCascade()`.

This comparison is the key precision win of `abi` over `graph`. The cost of the comparison
is two `String.equals()` calls per recompiled class.

## Binary representation vs string canonical form

### Design verdict: String form is correct

- The canonical string is **never persisted**. Data flow: `classfile bytes →
  buildCanonicalForm() → String (transient) → Sha256.hash() → 16-char hex fingerprint`.
- String allocation cost per class: ~200–800 chars. For 1000 classes: ~500KB garbage + ~1ms
  encoding. Not measurable against classfile I/O.
- The string representation serves as a diagnostic tool for debugging fingerprint diffs.

### When to revisit

If `buildCanonicalForm()` is touched for another reason (e.g., adding annotation values,
handling records/sealed classes), switch to streaming binary at that point. Keep a
`buildCanonicalString()` for debug logging.

## Parallelizing analysis

### Design verdict: No for incremental rounds; defer for full builds

**Incremental round** (hot path — 1–5 sources, 3–15 class files):
- Thread pool overhead exceeds the work.
- The round is already gated behind javac — analysis cannot overlap with compilation.

**Full build** (cold path — all sources compiled in one round):
- Embarrassingly parallel (no inter-class dependencies during analysis).
- BUT: Maven already parallelizes at the module level (`-T`).
- Concurrency surgery required on `results` (`LinkedHashMap`), `outputClassIndex`,
  and `state.setType()`.

### Real performance lever

`BytecodeAnalyzer.analyze()` is called even in `graph` mode, though only dep-union and
metadata are needed (not sig/impl split or fingerprint). A lighter constant-pool-only scan
for graph mode could be 3–5× faster per class — more impactful than parallelism.

## ABI manifest design: two-file split

`IncrementalState` (binary, `target/`) and `abi-fingerprints` (text, `target/`) are
deliberately separate:

- `IncrementalState` is private incremental bookkeeping for the current module — binary,
  versioned, not meant for external readers.
- `abi-fingerprints` is the **public API boundary** between modules — text, minimal
  (type → fingerprint only), intentionally readable and forward-compatible across plugin versions.

Merging them would conflate private bookkeeping with the public inter-module interface.
Downstream modules should never need to load the full upstream `IncrementalState`.

## `isPrivateOrSynthetic()` — critical invariant

`collectTypes()` must use `isPrivateOrSynthetic()` for **both** fields and methods.
Bridge methods (generated by javac for generics, covariant returns) are excluded from the
canonical ABI form, but if only `isPrivate()` is checked, their descriptor types falsely
appear in `signatureTypes` as API-surface dependencies.

## module-info fingerprinting

A `module-info` class file without a `Module` attribute must return a deterministic sentinel
fingerprint — `Sha256.hash("module-info:no-module-attribute")`. Returning an empty fingerprint
would make any two such classes appear ABI-identical, suppressing cascade when they differ.
