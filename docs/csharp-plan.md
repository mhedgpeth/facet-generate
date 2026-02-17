# C# Emitter Implementation Plan

## Context

Add C# code generation support to facet-generate, following the Kotlin emitter as the reference implementation. This enables using `facet generate` to produce C# types for the People Work Windows release.

## BLOCKER: Depends on `swift-emitter-v2` merge

The `upstream/swift-emitter-v2` branch (redbadger/facet-generate) introduces `WithEncoding<T>` and `Container<'a>` wrapper types and migrates Kotlin to use fully qualified `Emitter` trait syntax. This is **required** before we can add `Emitter<CSharp>` — without it, implementing the trait for the same types as Kotlin causes compile-time ambiguity errors on all `.write()` calls.

**Status**: Waiting for `swift-emitter-v2` to merge to `main`. Once merged, rebase this branch and proceed with Phase 1.

**Contact**: Reach out to redbadger maintainers about merge timeline.

## Decisions

- **C# records** (not classes) — immutable, auto Equals/GetHashCode/ToString, closest analog to Kotlin data classes
- **Target .NET 8+** (LTS) — supports records, file-scoped namespaces, Int128/UInt128 natively
- **JSON serialization via System.Text.Json** — the standard .NET JSON library, analogous to kotlinx.serialization
- **Bincode/BCS out of scope initially** — would require a C# serde runtime

## Hazards Identified

### 1. The `emit!` macro will break if we add C#

The generic `emit!` macro (`src/lib.rs:20`) calls `($encoding, item).write(&mut w)`. This relies on there being **exactly one** `Emitter<L>` impl for `(Encoding, (&QualifiedTypeName, &ContainerFormat))`. Currently only Kotlin implements this. If C# also implements it, Rust can't resolve which `L` to use → compile error.

**Solution**: C# emitter tests use a dedicated helper function (not the `emit!` macro), similar to how Java/Swift have their own `emit_java!`/`emit_swift!` macros with explicit types.

### 2. C# naming conventions differ from Kotlin

- Properties: **PascalCase** (not camelCase) → use `heck::ToUpperCamelCase`
- Enum variants: **PascalCase** (not SCREAMING_CASE)
- Namespace: file-scoped `namespace Example;` (not `package com.example`)

### 3. Generation tests require an Installer

The `src/generation/tests/basic/mod.rs` tests use `SourceInstaller` to write files to a temp dir. We need at minimum a basic Installer that writes `.cs` files to the right paths, even if we skip manifest generation.

### 4. Not all test modules test all languages

The `test!` macro in `src/tests/*/mod.rs` — each module lists which languages it covers. Some only test `kotlin`. We add `csharp` incrementally, not blindly to all 77 test modules.

## Type Mapping: Rust → C#

| Rust / Format | C# | Notes |
|---|---|---|
| Unit | object | C# has no Unit type; use object for now |
| Bool | bool | |
| I8 / U8 | sbyte / byte | |
| I16 / U16 | short / ushort | |
| I32 / U32 | int / uint | |
| I64 / U64 | long / ulong | |
| I128 / U128 | Int128 / UInt128 | .NET 8+ |
| F32 / F64 | float / double | |
| Char | char | C# has a native char type |
| Str | string | |
| Bytes | byte[] | |
| Option\<T\> | T? | nullable |
| Seq\<T\> | List\<T\> | System.Collections.Generic |
| Set\<T\> | HashSet\<T\> | |
| Map\<K,V\> | Dictionary\<K, V\> | |
| Tuple(2) | (T1, T2) | C# value tuples |
| Tuple(3) | (T1, T2, T3) | |
| Tuple(N) | (T1, ..., TN) | |
| TupleArray { size, content } | List\<T\> | Same as Kotlin approach |
| TypeName | Qualified class reference | |

## C# Code Patterns

### Struct → C# record (positional syntax)
```csharp
/// line 1
/// line 2
public record Child(
    string Name
);
```

### Empty struct → empty record
```csharp
public record UnitStruct();
```

### Enum (all unit variants) → C# enum
```csharp
/// This is a comment.
public enum Colors
{
    Red,
    Blue,
    /// Green is a cool color
    Green
}
```

### Enum (mixed variants) → abstract record hierarchy
```csharp
public abstract record MyEnum
{
    public record Unit() : MyEnum;
    public record NewType(string Value) : MyEnum;
    public record Tuple(string Field0, int Field1) : MyEnum;
    public record Struct(bool Field) : MyEnum;
}
```

### NewTypeStruct → record with single Value field
```csharp
public record Wrapper(string Value);
```

### TupleStruct → record with positional fields
```csharp
public record TupleStruct(string Field0, int Field1);
```

### JSON serialization (System.Text.Json)
```csharp
using System.Text.Json.Serialization;

[JsonDerivedType(typeof(ChildVariant), "Child")]
public abstract record Parent
{
    public record ChildVariant(
        [property: JsonPropertyName("value")] Child Value
    ) : Parent;
}
```

## Files to Create/Modify

### New files (Phase 1-2)
- `src/generation/csharp/mod.rs` — `mod emitter; mod generator; pub use generator::CodeGenerator;`
- `src/generation/csharp/emitter/mod.rs` — all `Emitter<CSharp>` impls
- `src/generation/csharp/emitter/tests.rs` — inline snapshot tests (Encoding::None)
- `src/generation/csharp/generator.rs` — `CodeGenerator` implementing `CodeGen<'a>`

### Modified files (Phase 1-2)
- `Cargo.toml` — add `csharp = ["indoc"]` to features, add to `generate` list
- `src/generation/mod.rs` — add `#[cfg(feature = "csharp")] pub mod csharp;` + `Language::CSharp` variant

### New files (Phase 3)
- `src/generation/csharp/emitter/tests_json.rs` — JSON encoding snapshot tests

### New/Modified files (Phase 4)
- `src/tests/mod.rs` — extend `test!` macro with `(@package csharp)` and `(@out csharp)`
- `src/tests/can_generate_unit_enum/mod.rs` — add `csharp` to test list (and other test modules)
- `src/tests/can_generate_unit_enum/output.cs` — expected snapshot (and others)

## Implementation Phases — Detailed

### Phase 1: Scaffold + Format type mapping + first passing test

**Goal**: `cargo test csharp` passes with one snapshot test (unit struct).

**Step 1.1** — Wire up the module:
- `Cargo.toml`: add `csharp = ["indoc"]` to `[features]`, add `"csharp"` to `generate` list
- `src/generation/mod.rs`: add `#[cfg(feature = "csharp")] pub mod csharp;` after typescript, add `CSharp` to `Language` enum + Display impl
- `src/generation/csharp/mod.rs`: create with `mod emitter; mod generator; pub use generator::CodeGenerator;`

**Step 1.2** — Create the emitter skeleton:
- `src/generation/csharp/emitter/mod.rs`:
  - `pub struct CSharp;` (marker type for the Emitter trait)
  - `impl Emitter<CSharp> for Format` — the full type mapping table above
  - `impl Emitter<CSharp> for Named<Format>` — emit `TypeName PropName` with PascalCase via `heck::ToUpperCamelCase`
  - `impl Emitter<CSharp> for Doc` — emit `/// comment` lines (same as Kotlin)
  - `impl Emitter<CSharp> for (Encoding, (&QualifiedTypeName, &ContainerFormat))` — dispatch to helper functions
  - Helper functions: `record()`, `enum_type()`, `abstract_record()` (analogous to Kotlin's `data_class()`, `enum_class()`, `sealed_interface()`)
  - `#[cfg(test)] mod tests;`

**Step 1.3** — Create the generator:
- `src/generation/csharp/generator.rs`:
  - `pub struct CodeGenerator<'a> { config: &'a CodeGeneratorConfig }`
  - Implement `CodeGen<'a>` trait (new + write_output)
  - `output()` method: create `IndentedWriter(Space(4))`, iterate registry, call `(encoding, item).write(w)` for each container
  - **No namespace handling initially** — just emit containers

**Step 1.4** — Create the first test:
- `src/generation/csharp/emitter/tests.rs`:
  - Create helper: `fn emit_csharp(encoding: Encoding, registry: &Registry) -> String` that iterates registry and calls `(encoding, item).write(w)` — avoids the `emit!` macro
  - Write `unit_struct` test with `insta::assert_snapshot!`

**Step 1.5** — Run `cargo test csharp` and iterate until it passes.

### Phase 2: All container formats + emitter tests

**Goal**: Complete emitter test coverage matching Kotlin's `tests.rs`.

**Step 2.1** — Implement and test each container format:
1. UnitStruct → `public record Name();`
2. NewTypeStruct → `public record Name(Type Value);`
3. TupleStruct → `public record Name(Type Field0, Type Field1);`
4. Struct (with fields) → `public record Name(Type Prop1, Type Prop2);`
5. Enum (all unit) → `public enum Name { V1, V2 }`
6. Enum (mixed) → `public abstract record Name { ... nested records ... }`

**Step 2.2** — Test all Format variants:
- Primitive types (all numeric types, bool, char, string)
- Collections (Vec, HashMap, HashSet, BTreeMap, BTreeSet)
- Option (nullable with `?`)
- Tuples (2-tuple through N-tuple using C# value tuples)
- TupleArray (List<T>)
- Smart pointers (Box, Rc, Arc — transparent, unwrapped during reflection)
- Bytes (`byte[]`)
- Nested generics (`Option<Vec<Map<...>>>`)

**Step 2.3** — Test user-defined type references.

Each test follows the Kotlin pattern: define Rust type with `#[derive(Facet)]`, call helper, `insta::assert_snapshot!`.

### Phase 3: JSON Serialization

**Goal**: `Encoding::Json` produces correct System.Text.Json attributes.

**Step 3.1** — Module preamble with JSON:
- `Emitter<CSharp> for Module` — emit `using System.Text.Json.Serialization;` when encoding is JSON

**Step 3.2** — Record with JSON:
```csharp
public record Child(
    [property: JsonPropertyName("name")] string Name
);
```

**Step 3.3** — Enum with JSON:
```csharp
[JsonConverter(typeof(JsonStringEnumConverter))]
public enum Colors { Red, Blue, Green }
```

**Step 3.4** — Abstract record with JSON (discriminated union):
```csharp
[JsonDerivedType(typeof(Child), "Child")]
[JsonDerivedType(typeof(UnitVariant), "UnitVariant")]
public abstract record Parent { ... }
```

**Step 3.5** — Create `tests_json.rs` with snapshot tests for each pattern.

### Phase 4: Generator (namespace/qualified name handling)

**Goal**: Multi-module output with correct C# namespace references.

**Step 4.1** — Implement `update_qualified_names` in generator (parallel to Kotlin's):
- Root namespace → current module name
- Named namespace → `module.namespace`
- External packages → external path handling
- C# uses dot-separated namespaces like Kotlin packages, so this logic is nearly identical

**Step 4.2** — Module preamble:
- Emit `namespace ModuleName;` (file-scoped)

**Step 4.3** — Test with `emit_two_modules!` macro (this already exists and is generic).

### Phase 5: Unit tests via `test!` macro

**Goal**: C# participates in the `src/tests/` snapshot test suite.

**Step 5.1** — Extend `test!` macro in `src/tests/mod.rs`:
```rust
(@package csharp) => { "Example" };
(@out csharp) => { "output.cs" };
```

**Step 5.2** — Add `csharp` to a subset of test modules and create `output.cs` snapshots:
- Start with `can_generate_unit_enum`, `can_generate_unit_structs`
- Expand to more modules as confidence grows

### Phase 6: dotnet compilation validation

**Goal**: Prove generated C# actually compiles.

**Step 6.1** — Create a scratch dotnet project that references generated output.
**Step 6.2** — Generate C# for sample types, copy into project, run `dotnet build`.

## Verification

1. `cargo test` — all existing tests continue to pass (no regressions)
2. `cargo test csharp` — new C# emitter tests pass
3. `dotnet build` — generated C# compiles successfully

## Out of Scope (for now)
- C# serde runtime library (Bincode/BCS encoding)
- Installer / SourceInstaller impl (file layout, .csproj generation)
- NuGet package manifest generation
- Generation tests (`src/generation/tests/basic/`) — these require an Installer
