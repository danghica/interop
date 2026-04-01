# Cangjie `Extern` type (proposal)

**Status.** This document is a **language design proposal**: the core `Extern` type described here is **not** part of current shipping Cangjie. Sections that refer to **built-in `Any`** or tooling describe **today’s** Cangjie unless marked otherwise.

# Design rationale

The language design **in this document** will replace the following style of interop with ArkTS. The left-hand snippet is **illustrative pseudocode** (not a single guaranteed API shape):

```cangjie
// call a function named Add(2, 3) from ArkTS
let addF = api["Add"].asFunction()
let args: Array<JSValue> = [
    runtime.number(2).toJSValue(),
    runtime.number(3).toJSValue(),
]
let receiver: JSValue = runtime.null().toJSValue()
let result = addF.call(args, thisArg: receiver)
    .asNumber().toInt32()
```

To this new style:

```cangjie
// the type of api.Add() : (Int32, Int32) -> Extern
let result: Int32 = api.Add(2, 3)
```

# Lexical

- New proposed keyword: `Extern`

# Types

- New proposed type: `Extern`

## Extern type

`Extern` is a special type given to values which originate in foreign code. This can happen in the following situations:

- Cangjie code calls a foreign function, which can return a value of `Extern` type.
- Foreign code calls a Cangjie function, which can bind the arguments of the latter to `Extern`.

Cangjie programs can **propagate** extern values as *unopened envelopes* (pass them as arguments, return them, store them in **native**-typed slots only as permitted below) but must not **eliminate** them through native operations until conversion. **Normatively**, the following *elimination forms* on values of non-native data type (or on `Extern` alone) are **rejected at type-checking**:

- **Direct boolean guards:** the condition expression in `if (e)`, `while (e)`, and similar control forms must be typed as `Bool` (after the usual rules); `Extern` and non-native data containing `Extern` are not allowed in that position without prior conversion to a native type.
- **Field and component projection** that would expose a non-native field type as a usable value in a forbidden context (aggregate rules follow `~ext` and binding rules).
- **Arithmetic and user-defined operators** whose parameters are not extern-compatible with the operand types required by the operator.
- **Pattern matching** that inspects `Extern` or non-native structure beyond what the type system allows at boundaries.

Operations that only **forward** an `Extern` (function call, return, assignment into a slot that type-checks via extern-compatible conversion to a **native** type) are **propagation**, not **elimination**, in this sense.

In order to enable eliminated use, extern values must first be **implicitly** or **explicitly** converted to a native Cangjie type at the boundary. That conversion may fail at **runtime** if the foreign value cannot match the target type.

Informally, any Cangjie **data** type that does not use `Extern` in its definition is said to be *native*. Any Cangjie **data** type that uses `Extern` is said to be *non-native*. For example the following types are non-native: `(Bool, Extern)`, `Array<Extern>`, `Option<Array<Extern>>`. **Function types are always native** (for binding and storage), including when parameter or result types mention `Extern` (for example `Extern -> Int32`, `(Int32, Int32) -> Extern`). A field or variable whose declared type is a **function** type—say `(Extern) -> Extern`—is therefore allowed, because that declared type is native even though it mentions `Extern` inside the function type.

Binding declarations for variables and fields must have *native* types (native data or any function type). Function *signatures* may still use non-native **data** in parameters and results.

The following examples produce **type errors** (compile time):

```cangjie
let x: Extern = e            // illegal even if expression e has Extern type
var y: Extern                // illegal even if y is left uninitialized
let z: (Extern, Int32) = e   // illegal because the variable type is non-native
let w = e                    // illegal because the type of w is inferred as Extern
```

It is legal for function parameters to have non-native types, and also for functions to return non-native types. For instance the identity function works on extern values:

```cangjie
func id(x: Extern): Extern { return x }
```

### Assignment and non-native assignees

In assignment, the **assignee** (the mutable location written to) must have a **native** type. For instance:

```cangjie
func foo(x: Array<Extern>, y: Extern): Unit {
    x[0] = y   // type error: the element slot has type Extern, which is non-native data
}
```

### `Array<Extern>` and other non-native parameters: read, index, iterate

When a parameter has type `Array<Extern>` (or another non-native aggregate allowed in a signature):

- **Indexing** `x[i]` as an **r-value** has type `Extern` (or whatever the element type is). That is **propagation**, not assignment into a non-native slot. The resulting `Extern` is still subject to the **no-elimination** rules until conversion.
- **Length** and similar operations are allowed if their results are **native** (e.g. `Int64`); they do not inspect the extern payload.
- **Iteration** that yields elements of type `Extern` is allowed as pass-through only; each yielded value obeys the same rules as any `Extern`.
- **Assignment** to `x[i]` or to any l-value whose slot type is non-native **data** remains **illegal** in Cangjie code; see [Foreign mutators for non-native storage](#foreign-mutators-for-non-native-storage).

### Foreign mutators for non-native storage

When foreign code owns a mutable buffer typed as `Array<Extern>` (or similar), updates that would require an illegal assignment in Cangjie may be performed by a **foreign** entry point that the implementation provides or that the user declares, for example:

```cangjie
// Declared with `foreign` (or equivalent); not implemented in Cangjie.
// Writes one element; semantics and threading are implementation-defined.
foreign func assignExternAt(a: Array<Extern>, i: Int64, v: Extern): Unit

func foo(x: Array<Extern>, y: Extern): Unit {
    assignExternAt(x, 0, y)
}
```

**Contract (informal):** the foreign implementation performs the write on the object bound to `x`; Cangjie’s type system does not model side effects on the `Extern` payload beyond acknowledging the call as an **FFI boundary**. Any number of similar `foreign` mutators are allowed under the same pattern.

If an expression has `Extern` type it can be **implicitly** converted when bound to a variable or assignee whose type is a **native** type **extern-compatible** with `Extern`; it can be **explicitly** cast with `as`. If `e1` is an expression of type `Extern`, the following are legal:

```cangjie
let x: Int32 = e1
var y: Int32 = e1
y = e1               // y is already declared as Int32
let z = e1 as Bool   // z has type Option<Bool>
```

A **redundant** explicit cast is one where the target type is already uniquely determined by context (e.g. `as Bool` when the binding already requires `Bool` and the conversion is the same as the implicit boundary); such casts may still be allowed for documentation or symmetry.

Although legal, these forms may **fail at runtime** at the boundary if the foreign value does not convert to the target native type.

If a non-native data type and a native data type have the same outer shape except that some sub-trees (in the definition, syntactically) are just `Extern` on the left and native on the right, the left is said to be *extern-compatible* to the right; formally `T' ~ext T` as below. For instance `(Bool, Extern)` is extern-compatible to `(Bool, Int32)` and to `(Bool, Bool)`; it is also extern-compatible to `(Bool, (Bool, Int32))` because `Extern` (leaf) is extern-compatible to the sub-tree `(Bool, Bool)`. Top-level `Extern` is compatible with any native data type (`Extern ~ext T` for native data `T`).

**Subtyping.** Extern-compatibility `~ext` is **not** Cangjie subtyping (`<:`). In general neither `Extern <: Int32` nor `Int32 <: Extern` holds. Conversions use **`~ext`** and explicit rules at bindings, assignments, casts, and calls—not the ordinary subtype lattice. That avoids confusing `Extern` with a universal supertype or with `Any`.

Formally, `~ext` is the least relation on data types generated by (right-hand side always native data):

- **Universal replacement:** `Extern ~ext B` for every native data type `B`.
- **Reflexivity on native data types:** if `T` is native data, then `T ~ext T`.
- **Data-type constructors:** for every **admitted** type constructor `C` (tuples, `Option`, `Array`, standard library and user-defined **data** generics, subject to variance and constraints in the full language), if `Ai ~ext Bi` for all type arguments, then `C<A1, ..., An> ~ext C<B1, ..., Bn>`. **Function types** are not introduced by this clause; they remain native as a whole per the rule above. 

The universal replacement rule models a raw foreign value that may decode to any native data shape at a boundary, including composite.

Treating all function types as native does not weaken safety: boundary failures remain confined to explicit conversion sites (casts, extern-compatible bindings and assignments for *data*, and the usual checks at *application* when a parameter or result type mentions `Extern`). The predicate “native” for storage is separate from “contains no `Extern` anywhere”; only the latter would characterize “purely foreign-free” types, which this spec does not use as the storage rule.

Examples:

```cangjie
Extern ~ext Bool                          // true
Extern ~ext (Bool, Bool)                  // true
(Extern, Bool) ~ext (Bool, Bool)          // true
(Extern, Bool) ~ext (Bool, (Bool, Bool))  // false
(Extern, Bool) ~ext (Int32, Int32)        // false
```

The illegal and legal examples below generalize to types that are not extern-compatible and types which are extern-compatible, respectively. Suppose that `e` is an expression of type `(Extern, Bool)`. A cast to `Bool` is **not** justified by `~ext` on the tuple as a whole; use conversion at a **native** target shape that is extern-compatible (for example bind to `(Int32, Bool)` or convert the first component after a separate rule for projection, if the language adds one). For illustration of **`as` on plain `Extern`**, let `e1` be of type `Extern`; then `let z: Option<Bool> = e1 as Bool` is legal. A redundant explicit cast repeats the same boundary check (e.g. `e1 as Bool` when context already forces conversion to `Bool`).

A non-native data type may never appear as the **target** of a type cast. For instance the following are **type errors** (or rejected forms), not parse errors:

```cangjie
e as Extern
e as (Bool, Extern)
e as Array<Extern>
```

The following expressions are **type errors** because they are not extern-compatible to `(Extern, Bool)`:

```cangjie
let x: Int32 = e           // illegal
var y: (Bool, Int32) = e   // illegal
var z: (Bool, Bool, Bool)
z = e                      // illegal
```

The following expressions are **correct**:

```cangjie
let x: (Int32, Bool) = e
var y: ((Bool, Bool), Bool) = e
var z: (Bool, Bool)
z = e
```

## Rejected alternative (not in this spec)

An earlier **design option** allowed contexts that **force** a unique native type—such as `&&` requiring `Bool`—to accept operands of type `Extern` alongside `Bool`, so that `e1 && e2` might type-check when both are `Extern`. That **conflicts** with the normative rule that **direct boolean elimination** (including short-circuit operators whose result is `Bool`) requires proper conversion, and it would complicate inference (what if both operands are `Extern` but the intended conversion differs?). **This document does not adopt that option.**

## Open issues

- **Runtime checks for boxed generics:** feasibility of enforcing `~ext` at runtime when generic code erases types (e.g. boxed representations) needs a separate analysis.
- **Weaker storage rules:** if future revisions allow more non-native bindings, the assignment and foreign-mutator story must be updated so boundary failures stay explicit.

# Safety property

Type safety with explicit boundary failure.

**Setting.** Consider a **closed** program (fixed set of modules) where **FFI** is limited to declared **foreign** functions and built-in boundary APIs. **Heap** and **control stack** follow the language’s usual operational rules; only **extern-to-native conversion** at specified sites may fail with a **boundary-conversion error**. Other failures (uncaught exceptions, resource errors) are grouped under “throws or diverges” below.

For any closed, well-typed **expression or program fragment** whose **declared** type is a **native** type (including the result type of `main` or of an exported function), every **maximal** execution has exactly one of the following outcomes:

1. The execution **diverges**, or **throws** a normal language exception (or otherwise aborts outside boundary conversion).
2. The execution **terminates normally** with a value of the declared **native** result type.
3. The execution terminates with a **runtime boundary-conversion error** at an **explicit** extern-to-native boundary (implicit or explicit cast, extern-compatible binding, or `foreign` call that performs conversion).

Equivalent stuck-state formulation:

- A well-typed program cannot get stuck in an unclassified state.
- If execution reaches a non-reducible non-value state, that state is either a failure of the underlying runtime or a **declared boundary-conversion failure**.

Unlike the usual `dynamic` type used by C#, which **suppresses** type inference with dangerous consequences, the approach here:

- cannot camouflage a badly-typed program (in the sense of non-camouflage below),
- cannot replace normal type inference in code that does not touch `Extern`,
- cannot be used as a hack for flow-sensitive typing.

More formally:

**Non-camouflage (design goal):** The extended system should not admit a typing derivation that accepts a program **only** by widening positions to `Extern` (or to boundary types) when the same program would be rejected if every `Extern`-origin position were replaced by a **fixed** native type consistent with how the programmer intends to use that boundary. (A full proof obligation is implementation- and rule-specific.)

**Inference conservativity:** For an expression whose typing derivation mentions **no** `Extern` except possibly in subexpressions that are **direct operands** of a declared **FFI boundary**—a `foreign` call, a built-in interop primitive, or an expression explicitly annotated as boundary-consuming—the **inferred** types match the baseline Cangjie rules, and no **new** ambiguous overload resolution arises **except** where `Extern` or interop APIs introduce genuine ambiguity that the programmer must resolve.

# Extern vs Any vs Dynamic

This section records how the proposed `Extern` type relates to Cangjie’s built-in `Any` and to a **hypothetical** `Dynamic` type modeled on C# `dynamic` (Cangjie does not provide such a type today). `Any` is summarized below; `Dynamic` is discussed in [Comparison with a C#-style Dynamic](#comparison-with-a-c-style-dynamic-hypothetical). Informal contrast with real C# `dynamic` also appears under [Safety property](#safety-property).

## Comparison with `Any`

`Any` is as described in current Cangjie documentation: the empty interface `interface Any {}`, with all interfaces extending it and all non-interface types implementing it, so `T <: Any` for every type `T`. Values of type `Any` are ordinary Cangjie values; recovering a concrete type uses `as`, which yields an `Option`.

| | **Any** | **Extern** (this proposal) |
|---|---------|----------------------------|
| **Primary role** | Universal supertype for heterogeneous *native* Cangjie values | Marker for values that *originate in foreign code*; opaque until converted to a native type |
| **Typical use** | Containers or APIs that intentionally erase to the top of the Cangjie lattice (e.g. `var any: Any = 1; any = "hello"`) | FFI parameters, results, and wrappers; consume via extern-compatible binding or explicit cast to native types |
| **Operations before narrowing** | No extra “envelope” rules beyond normal typing; still need `as` or APIs to use as `Int32`, a class, etc. | Must not eliminate `Extern` via guards, arithmetic, pattern matching, etc. until converted (see [Extern type](#extern-type)) |
| **Storage (this proposal)** | Normal rules; `Any` is a standard interface type | Locals and fields use *native* types only; no binding annotated or inferred as bare `Extern`, etc.; assignment targets must be native |
| **Type inference** | Part of the baseline subtype hierarchy | Intended not to subsume or break inference the way C# `dynamic` can (see [Safety property](#safety-property)) |
| **Interop tooling** | For example, TypeScript `any` is mapped to Cangjie `Any` in dts2cj-style generated code, with bridge helpers such as `Any.fromJSValue` / `toJSValue` | Intended to replace verbose “everything is `JSValue` + `asFunction()` / `asNumber()` …” patterns with surface types that still fail at explicit boundaries |

In short: **Any** is maximal polymorphism inside the Cangjie type system; **Extern** is provenance and enforced boundary conversion for foreign values.

## Comparison with a C#-style `Dynamic` (hypothetical)

Cangjie has no `Dynamic` type. For comparison, imagine a type **Dynamic** designed like C# **dynamic**: a static type that **defers** member lookup, operators, and conversions to **runtime**, via compiler-generated binding (as in C#’s runtime binder), so that expressions involving `Dynamic` are type-checked only to the extent the binder can succeed at run time.

| | **Hypothetical Dynamic (C#-style)** | **Extern** (this proposal) |
|---|-------------------------------------|----------------------------|
| **Intent** | Erasure of static typing in a **region** of code: call arbitrary “members,” apply operators, and pass values through APIs as if dynamically typed | Mark **foreign-origin** values and forbid treating them as native until an **explicit** extern-to-native conversion |
| **Member access / operations** | Typically **allowed** at compile time; failures are **runtime** (e.g. missing member, bad argument types) | **Disallowed** on bare `Extern` until converted; no open-ended “try any operation” surface |
| **Provenance** | Not tied to FFI: any value could be lifted to `Dynamic` | Specifically **boundary** values from foreign code |
| **Inference and checking** | **Suppresses** normal static rules in `Dynamic`-typed subexpressions; can let ill-typed-looking programs compile and fail later (see [Safety property](#safety-property)) | **Non-camouflage** and **inference conservativity**: cannot “fix” a program by widening to `Extern`; native-only code keeps baseline inference |
| **Failure locus** | Wherever the runtime binder executes an operation | Declared **boundary conversions** (casts, extern-compatible bindings, application at `Extern`-bearing types) |

**Can Dynamic replace Extern?** Not as specified here. A C#-style `Dynamic` is a **different mechanism**: late binding everywhere the type flows, not a sealed envelope with **only** conversion sites as checks. Conversely, **Extern** does not replace **Dynamic**: it does not offer arbitrary late-bound member calls on native or foreign values inside Cangjie code.

## Can `Any` replace `Extern` (or the reverse)?

**Not in general—they are complementary.**

1. **Extern** does not subsume **Any**. Use cases that need the universal supertype for *native* values (heterogeneous collections, APIs that erase to the top type, generated mappings from TS `any`) are modeled by `Any`. `Extern` does not mean “any Cangjie type”; it means “value from another runtime until decoded to native.” Replacing `Any` with `Extern` would mis-model purely native code and would clash with the placement rules in this document (e.g. forbidding `Extern` locals).

2. **Any** does not subsume **Extern**. Every native value is already an `Any`, so **Any** does not mark foreign provenance or enforce the “no elimination until native conversion” discipline described for `Extern`. For JS and similar runtimes, truly untyped foreign payloads are often represented today with **JSValue** (or analogous bridge types); this proposal’s `Extern` is a core-language boundary contract at that layer, not the same abstraction as `Any`.

**Summary:** `Any` answers “hold any Cangjie-typed value uniformly”; `Extern` answers “this crossed from foreign code—only treat it as native at explicit conversion sites.” Neither is a drop-in substitute for the other. A hypothetical C#-style `Dynamic` answers “defer static rules and bind at runtime”; that is again a different axis from both `Any` and `Extern`.
