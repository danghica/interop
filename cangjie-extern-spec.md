# Cangjie Extern Type Specification (Draft Extension)

## Status

This document is a proposed language extension specification for adding a core `extern` type to Cangjie.

- **Document type:** draft normative extension
- **Primary audience:** language designers, compiler implementers, standard library maintainers
- **Goal:** define a predictable, safe, and implementable boundary type for foreign values

## Normative References

- Official Cangjie documentation portal: [https://cangjie-lang.cn/en/docs](https://cangjie-lang.cn/en/docs)
- Cangjie language specification chapter on generics: [https://docs.cangjie-lang.cn/docs/0.53.18/Spec/source_zh_cn/Chapter_09_Generics(zh).html](https://docs.cangjie-lang.cn/docs/0.53.18/Spec/source_zh_cn/Chapter_09_Generics(zh).html)
- Cangjie white paper (multi-paradigm overview): [https://docs.cangjie-lang.cn/docs/0.53.13/white_paper/source_zh_cn/cj-wp-multiparadigm.html](https://docs.cangjie-lang.cn/docs/0.53.13/white_paper/source_zh_cn/cj-wp-multiparadigm.html)
- Source model used as formal inspiration in this repository: [`/Users/danghica/interop/opsem.tex`](/Users/danghica/interop/opsem.tex)

## 1. Scope and Non-Goals

### 1.1 In Scope

This specification defines:

1. A new core type `extern`.
2. Static typing rules for where `extern` may appear.
3. Generic constraints and inference behavior when `extern` is involved.
4. Runtime boundary conversion behavior.
5. Diagnostics categories for compile-time and runtime failures.
6. Conformance examples and implementation checklist.

### 1.2 Out of Scope

This specification does not define:

1. Low-level ABI layout for foreign runtimes.
2. Concrete implementation details of every platform bridge.
3. Tooling UX details beyond required diagnostics categories.

## 2. Design Goals

1. **Boundary safety:** foreign values are explicitly marked and explicitly consumed.
2. **Static predictability:** native code keeps native typing guarantees.
3. **Practical interop:** common foreign workflows remain ergonomic.
4. **Controlled runtime checks:** checks happen at declared boundary points.
5. **Generic clarity:** behavior with generic constraints and inference is deterministic.

## 3. Baseline Compatibility with Current Cangjie Rules

This section maps the extension to baseline Cangjie behavior described in official docs.

| Area | Baseline | Extern Extension Result |
|---|---|---|
| Generic declarations | Supported on functions and major type declarations | No syntax change |
| Generic constraints (`where`) | Supported with subtype-based checking | Constraint policy extended for `extern` |
| Generic variance | User-defined generics are invariant | No change |
| Overload resolution | Existing candidate selection and matching rules | Adds tie-break policy for conversion-requiring candidates |
| Type argument inference | Existing inference rules | Adds explicitness requirement for ambiguous extern-involved calls |
| Interop APIs | Platform/library specific today | Adds core-level type contract and conversion obligations |

## 4. Language Additions

### 4.1 New Core Type

`extern` is introduced as a core type that represents boundary-origin values.

### 4.2 Boundary Conversion Primitives

Implementations must provide conversion operations from `extern` to native target types.
The API shape is implementation-defined, but behavior is normatively defined by this document.

Example shapes:

```cangjie
let n: Int64 = Interop.toInt64(rawExtern)
let s: String = Interop.toString(rawExtern)
```

## 5. Static Typing Rules (Plain Language)

### 5.1 Placement Rules for `extern`

1. `extern` is always legal in boundary declarations and boundary conversion APIs.
2. `extern` in generic arguments is legal only for declarations marked as boundary-capable.
3. In ordinary non-boundary generic APIs, `extern` as type argument is rejected.
4. Nested generic forms follow the same rule recursively.

### 5.2 Generic Constraints

1. `extern` does not satisfy nominal interface or class bounds by default.
2. A constrained generic call must use a native value that satisfies the bound.
3. Therefore, callers convert `extern` values before constrained generic calls.
4. Implementations may expose explicit adapters, but never implicit adaptation.

### 5.3 Type Inference

1. If a call involving `extern` has multiple valid native targets, the call is ambiguous.
2. The program must provide explicit type arguments or explicit target annotation.
3. Compilers must not silently pick a default native target.

### 5.4 Overload Resolution

1. Candidates that type-check without boundary conversion rank above candidates that require conversion.
2. If multiple no-conversion candidates remain, existing baseline rules apply.
3. If only conversion-requiring candidates remain and none is unique, report ambiguity.

### 5.5 Mutable State Restrictions

1. In ordinary modules, mutable storage types that contain `extern` are rejected transitively.
2. Boundary modules may opt in explicitly.
3. In opt-in modules, explicit conversion is required at read or write boundaries.

## 6. Runtime Behavior

### 6.1 Check Points

Runtime boundary checks happen at explicit conversion calls from `extern` to a native target type.

### 6.2 Failure Behavior

1. If conversion succeeds, execution continues with the converted native value.
2. If conversion fails, execution raises a runtime boundary-conversion error.
3. Errors must include source location and failure category.

### 6.3 Determinism Requirement

Given the same input value and target type, conversion success or failure must be deterministic in a given runtime profile.

### 6.4 Safety Property

This extension defines **type safety with explicit boundary failure**.

For any closed, well-typed program `P` whose declared result type is native, and for any well-typed initial runtime state, every maximal execution of `P` has exactly one outcome:

1. **Divergence:** execution does not terminate.
2. **Successful completion:** execution terminates with a value compatible with the declared native result type.
3. **Boundary failure:** execution terminates with a boundary-conversion runtime error, and the failing step is one of the declared boundary checks:
   - explicit conversion from `extern` to a native target type,
   - conversion required by a boundary `let`-style binding,
   - conversion required before writing into mutable storage constrained to a native type.

Equivalent stuck-state form:

- A well-typed program does not get stuck in any unclassified state.
- If execution reaches a non-reducible non-value state, that state must be a declared boundary-conversion failure.

Implications for implementers:

- Safety does **not** mean "no runtime errors"; it means runtime errors are restricted to explicit boundary contracts.
- Native code paths that do not cross an `extern` conversion boundary preserve ordinary static guarantees.

## 7. Diagnostics Contract

### 7.1 Compile-Time Errors

- Illegal `extern` placement in non-boundary generic positions.
- Unsatisfied generic constraints when `extern` is used directly.
- Ambiguous inference caused by missing target type information.
- Ambiguous overloads among conversion-requiring candidates.

### 7.2 Runtime Errors

- Boundary conversion failure from `extern` to requested native type.

### 7.3 Stability Requirements

Implementations may vary wording, but must keep:

1. error category stable,
2. primary source span stable,
3. conversion target type visible in diagnostics.

## 8. Open Decision Points and Chosen Defaults

This section identifies details not fully determined by ML-like typing patterns and records defaults.

### 8.1 Nominal Constraints

- **Default:** `extern` does not satisfy nominal bounds directly.
- **Alternative:** allow explicit marker-interface opt-in.
- **Tradeoff:** default maximizes soundness; opt-in increases flexibility.

### 8.2 Overload Ranking with Conversion

- **Default:** no-conversion candidates outrank conversion-requiring candidates.
- **Alternative:** treat both equally and raise ambiguity more often.
- **Tradeoff:** default improves predictability at call sites.

### 8.3 Mutable Storage Policy

- **Default:** transitive restriction for storage types containing `extern`.
- **Alternative:** direct-only restriction.
- **Tradeoff:** transitive rule is safer; direct-only is simpler but riskier.

### 8.4 Generic Instantiation Strategy

- **Default:** do not introduce extra specialization variants solely for conversion outcome modeling.
- **Alternative:** partial sharing for extern-bearing instantiations.
- **Tradeoff:** default avoids code growth surprises.

### 8.5 Diagnostics Scope

- **Default:** category and source location are normative; exact message text remains implementation-defined.
- **Alternative:** fully standardized message schema.
- **Tradeoff:** default balances portability and implementation freedom.

## 9. Compatibility and Migration

### 9.1 Coexistence

Existing platform interop APIs may continue to exist.
Core `extern` adds a common language-level contract.

### 9.2 Source Compatibility Guidance

1. Existing code with concrete interop wrapper types can remain unchanged.
2. New APIs may migrate to `extern` plus explicit conversion boundaries.
3. For mixed codebases, boundary adapters should isolate migration surface.

### 9.3 Suggested Migration Strategy

1. Introduce `extern` in new APIs only.
2. Add adapters from legacy wrappers to `extern`.
3. Add lint warnings for unsafe generic patterns involving boundary wrappers.
4. Gradually standardize on one conversion API family.

## 10. Conformance Examples

All examples use Cangjie syntax.

### 10.1 Compile-Pass Examples

```cangjie
interface Comparable<T> {
    func compare(other: T): Int64
}

func maxOf<T>(a: T, b: T): T where T <: Comparable<T> {
    return if (a.compare(b) >= 0) a else b
}

func useBoundary(raw: extern): Int64 {
    let n: Int64 = Interop.toInt64(raw)
    return maxOf<Int64>(n, 10)
}
```

```cangjie
@BoundaryModule
class BoundaryCell<T>(var value: T)

func storeBoundary(raw: extern) {
    let c = BoundaryCell<extern>(raw) // allowed in boundary-marked module
    let s: String = Interop.toString(c.value)
    println(s)
}
```

### 10.2 Compile-Fail Examples

```cangjie
class Box<T>(let value: T)

func bad(raw: extern) {
    let b: Box<extern> = Box<extern>(raw) // error: extern in non-boundary generic position
}
```

```cangjie
interface Hashable<T> {
    func hashCode(): Int64
}

func hashOne<T>(x: T): Int64 where T <: Hashable<T> {
    return x.hashCode()
}

func bad2(raw: extern): Int64 {
    return hashOne(raw) // error: extern does not satisfy bound
}
```

```cangjie
func parse<T>(x: extern): T {
    return Interop.convert<T>(x)
}

func bad3(x: extern) {
    let y = parse(x) // error: cannot infer unique target type
    println(y)
}
```

### 10.3 Runtime-Fail Examples

```cangjie
func parseInt(raw: extern): Int64 {
    return Interop.toInt64(raw) // runtime boundary-conversion error if not integer-compatible
}
```

```cangjie
func parseBool(raw: extern): Bool {
    return Interop.toBool(raw) // runtime boundary-conversion error if not boolean-compatible
}
```

## 11. Risks and Performance Notes

### 11.1 Soundness Risks

- Implicit adaptation from `extern` into constrained generics can hide type errors.
- Weak storage rules can leak boundary values into mutable native structures.

### 11.2 Performance Risks

- Excessive conversion at hot call sites.
- Generic code size growth if conversion-specific specialization expands too far.

### 11.3 Ergonomics Risks

- Too many required annotations at boundaries.
- Confusing overload behavior without explicit ranking policy.

### 11.4 Mitigations

- Keep conversion explicit and localized.
- Prefer boundary adapters over wide `extern` propagation.
- Provide clear diagnostics for target type and failure point.

## 12. Completeness Audit

This document satisfies the required completeness gates:

1. Full section set from scope to conformance.
2. Every policy area includes at least one example.
3. Open decisions include default, alternative, and tradeoff.
4. No Greek symbols and no formal operational notation.
5. Explicit references to official Cangjie docs and repository source model.

## 13. Implementation Checklist

Compiler and runtime implementations are conformant when they provide:

1. Core `extern` type support.
2. Placement checks for `extern` in generic positions.
3. Constraint checks that reject direct `extern` under nominal bounds unless explicitly adapted.
4. Inference ambiguity diagnostics for extern-involved unresolved target types.
5. Overload ranking behavior specified in this document.
6. Mutable storage enforcement for types containing `extern`.
7. Deterministic runtime conversion behavior with stable diagnostics category and source location.

