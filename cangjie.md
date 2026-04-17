# Cangjie `Extern` type (proposal)

**Status.** This document is a **language design proposal**: the core `Extern` type described here is **not** part of current shipping Cangjie. Sections that refer to **built-in `Any`** or tooling describe **today’s** Cangjie unless marked otherwise.

Below, *foreign* code means code in a dynamically or weakly typed language such as ArkTS, JavaScript, Python, and similar. This proposal does not address interoperability with C and Java, which already have dedicated support in the Cangjie specification.

# Design rationale

The language design **in this document** will make the following style of interop with ArkTS more ergonomic.

The snippet is **illustrative pseudocode** (not a single guaranteed API shape):

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

We aim to use this new style:

```cangjie
// declaring an external API class
external class API { ...
  func Add(x: Int32, y: Int32) : Extern
...
}

// use of an external function of the API
let result: Int32 = api.Add(2, 3)
```

The proposed approach is influenced by the use of `dynamic` in C# but with extra restrictions in order to prevent the known 'dark pattern' of using dyamic types to bypass the type system. 

# Lexical

- New proposed keywords: `external`,`Extern`

# Class and function modifier

The keyword `external` applies to individual `func` and `class` declarations whose implementations live outside Cangjie (for example in ArkTS). It marks an FFI surface: the compiler type-checks the Cangjie-side signature but does not check a body on the foreign side.

**Proposal (vs. C FFI):** Calls to `external` functions are intended to be ordinary safe calls at the Cangjie source level—callers do not wrap them in `unsafe`, unlike typical C interop in current Cangjie. Final rules depend on how this proposal is integrated with the shipping language.

For instance consider this class

```cangjie
external class Rectangle {
  var width: Float64
  var height: Float64
  init(w: Float64, h: Float64}
  func area(): Float64
  func color(): Extern
```

It declares an object that lives in the foreign memory space (e.g. an ArkTS virtual machine). These objects cannot be created using a constructor, so the following is a syntax error: 

```cangjie
let r = new Rectangle(1.0, 2.0)
```

Any use of the data or function members of an `external` class are instrumented by the compiler to import and export data from the foreign memory space. If the data types are Cangjie types (for instance in the example above `width: Float64`) they are always cast into that Cangjie type. If they have external type (e.g. `func color(): Extern`) then the rules for the `Extern` type apply, as described below.

# Types

- New proposed type: `Extern`

## Extern type

`Extern` is a special type given to values which originate in foreign code. This can happen in the following situations:

- Cangjie code calls a foreign function, which can return a value of `Extern` type.
- Foreign code calls a Cangjie function, which can bind the arguments of the latter to `Extern`.

Cangjie programs can **propagate** extern values as *unopened envelopes* (pass them as arguments and return them) but must not **eliminate** them through native operations until conversion. **Normatively**, any *elimination forms* on values of type `Extern` are **rejected at type-checking**. This also includes storing `Extern` values into local, global, or field variables.

Operations that only **forward** an `Extern` (function call and return) are **propagation**, not **elimination**, in this sense, so they are permitted.

**Propagation and elimination (informal).** *Propagation* means moving an `Extern` only by passing it as an argument, returning it, or otherwise forwarding it without treating it as native data. *Elimination* means using it as if it were already native: storing it in a `let`, `var`, or field as bare `Extern`, pattern-matching on it, applying native operators, indexing, and similar--anything that consumes the value *as* `Extern` rather than forwarding it unchanged.  

In order to use an extern value as ordinary native **data**, it must first be **implicitly** or **explicitly** converted at a **boundary**. That conversion may fail at **runtime** if the foreign value cannot match the target type. Binding `let x: Int32 = e` when `e` has type `Extern` is **not** elimination; it is **boundary conversion** to native `Int32` and is allowed. Storing bare `Extern` in a variable is forbidden precisely so every use as native data goes through an explicit boundary. 

The way conversion errors are handled at runtime are left unspecified at the language of language or compiler, and are delegated to the implementation. 

**Rules at a glance.** You may not bind `let`/`var`/fields to bare `Extern` or nest `Extern` inside tuple, array, option, or other generic **data** types. You may use `Extern` in function types and as parameter/result types. Generic type parameters cannot be instantiated as `Extern`. You cannot cast *to* `Extern`. Otherwise, pass and return `Extern` freely; convert to native types only at boundaries (bindings, assignments, and `as`), where failure is possible at run time. The normative bullets in the next subsection spell this out.

### Placement of `Extern` (normative)

- **Extern under data type constructors is forbidden.** It may not appear as a component of a tuple, as the argument of `Array<T>`, `Option<T>`, or any user-defined or standard-library **generic data** type, including nested positions (e.g. `Array<Array<Extern>>`, `(Bool, Extern)`). Any such written type is a **compile-time error**.
- **Extern inside function types is allowed.** Function types are **native as a whole** even when a parameter or result type is `Extern` or contains `Extern` only under further function arrows (e.g. `(Int32) -> Extern`, `Extern -> Int32`, `(Extern) -> Extern`).
- **Generic type parameters cannot be instantiated with `Extern`.** A type parameter `A` may not be replaced by `Extern`, explicitly or by inference, in any well-typed program. For example, `func foo<A>(x: Array<A>, y: A)` cannot be called at `A = Extern`.
- **`let` and `var` bound variables may not have `Extern` type.** Local, global, or field variables may not receive `Extern` explicitly or via type inference. 
- **Extern may never appear as the target of a type cast.** Values can only be converted *from* `Extern` to native data types.

### Native vs non-native data

- The **only** non-native **data** type is bare `Extern`. There are no composite non-native or heterogenous external-native data types: tuple, array, or option types that embed `Extern` are **ill-typed**, not a separate class of types.
- **Native data** means every well-formed **data** type other than `Extern`.

**Function types are always native**, including when parameter or result types mention `Extern`. For example:

- `Extern -> Int32`,
- `(Int32, Int32) -> Extern`.

A field or variable whose declared type is a **function** type, say `(Extern) -> Extern`, is therefore allowed, because that declared type is native even though it mentions `Extern` inside the function type.

### Examples

For instance, the following examples produce **type errors** (compile time):

```cangjie
let x: Extern = e            // illegal even if expression e has Extern type
var y: Extern                // illegal even if y is left uninitialized
let z: (Extern, Int32) = e   // illegal: tuples cannot contain Extern
let w = e                    // illegal, type of w is inferred as Extern
// illegal types (not allowed anywhere):
var a: Array<Extern>
var b: Option<Extern>
```

It is legal for function parameters and results to use `Extern` (the only non-native data type) and for function types to mention `Extern`. For instance the identity function works on extern values:

```cangjie
func id(x: Extern): Extern { return x }
```

It is also legal for a function to select between two `Extern` values

```cangjie
func sel(x: Extern, y: Extern): Extern { if (test()) { x } else { y } }
```

However, a function that takes two `Extern` values and returns them as a tuple is not possible:

```cangjie
func tup(x: Extern, y: Extern): (Extern, Extern) { (x, y) } // type error
```

### Foreign mutators and opaque handles

Homogeneous collections of foreign values cannot be expressed as `Array<Extern>` under this spec. When foreign code owns a mutable buffer of such values, updates that cannot be expressed as legal Cangjie assignments may be performed via a **foreign** entry point that takes an **opaque** FFI handle type (or `Array<Any>`, native wrapper, etc.), for example:

```cangjie
// Opaque type provided by foreign code
external func assignAtBuffer(arrayBuf: Extern, i: Int64, v: Extern): Unit

func useBuffer(arrayBuf: Extern, v: Extern): Unit {
    assignAtBuffer(arrayBuf, 0, v)
}
```

**Contract (informal):** The foreign side performs the write on the object denoted by the handle. Cangjie’s types do not track effects on the foreign payload except at this **FFI boundary**, so in-place mutation is modeled as happening in foreign space. If you need to mutate through ordinary Cangjie code paths, convert `arrayBuf` and `v` to native representations first.

### Boundary conversion to native data

If an expression `e` has type `Extern`, it may be **implicitly** converted when bound to a variable or assignee whose type is a **native data** type, or **explicitly** cast with `as`, at a **boundary** defined by the language and interop APIs. If `e1` is an expression of type `Extern`, the following are legal:

```cangjie
let x: Int32 = e
var y: Int32 = e
y = e               // y is already declared as Int32
let z = e as Bool   // z is inferred type Option<Bool>
```

**Subtyping.** Boundary conversion is **not** Cangjie subtyping (`<:`). In general neither `Extern <: Int32` nor `Int32 <: Extern` holds. Conversions are governed by explicit rules at bindings, assignments, casts, and calls, not the ordinary subtype lattice. That avoids confusing `Extern` with a universal supertype or with `Any`.

Which native targets are valid for a given foreign value, and how composite shapes are validated, is defined by the **implementation / interop API**. There is no separate structural “compatibility” relation on types. Model composite foreign values with **native** structs or classes, `Any`, extra parameters, several `Extern` parameters, or opaque FFI types, not `Extern` nested inside a data constructor.

A **redundant** explicit cast is one where `as` repeats a conversion the context already forces. For example, `let x: Option<Bool> = e as Bool` when the same conversion would apply implicitly to `let x: Option<Bool> = e`, if the language allows the latter. Note that the former is *safe* in the sense that if the cast fails it returns `Null`, whereas the latter is *unsafe* in the sense that if the conversion fails a runtime error is issued. 

Although legal, these forms may **fail at runtime** at the boundary if the foreign value does not convert to the target native type.

Treating all function types as native does not weaken safety: boundary failures remain confined to explicit conversion sites (casts, boundary bindings and assignments to native **data**, and checks at **call sites** where a parameter or result type is `Extern` or a function type mentioning `Extern`).

`Extern` may never appear as the **target** of a type cast, nor may a cast target be a **data** type that embeds `Extern` (such written types are already ill-formed). For instance the following are **type errors** (or rejected forms), not parse errors:

```cangjie
e as Extern
e as (Bool, Extern)   // ill-formed type
e as Array<Extern>    // ill-formed type
```

### Dynamic behavior

In addition to casting to a Cangjie type as discussed above an expression of `Extern` type can be used locally as a dynamic type via the member access operator (`.`). If `e: Extern` then `e.m` is valid for any identifier `m`. 

For instance consider interfacing with a JavaScript program that manages a DOM which we want to treat as an object for convenience, instead of making queries by label. 

```cangjie
func getDom(): External
let item: DomItem = getDom().body.menu.item
```

Note that unlike C# the following is not possible: 

```cangjie
func getDom(): External
let x: External = getDom().body.menu
let item: DomItem = x.item
```

# Safety property

We aim to preserve the type safety of Cangjie, and add explicit boundary failure.

Unlike the usual `dynamic` type used by C#, which **suppresses** type inference with dangerous consequences, the approach here:

- cannot camouflage a badly-typed program,
- cannot replace normal type inference in code that does not touch `Extern`,
- cannot be used as a hack for flow-sensitive typing.

More explicitly:

**Non-camouflage:** The extended system should not admit a typing derivation that accepts a program **only** by widening positions to `Extern` when the same program would be rejected if every `Extern`-origin position were replaced by a **fixed** native type consistent with how the programmer intends to use that boundary.

**Inference conservativity:** If Extern only appears where the language already treats something as an FFI boundary (not spread through the middle of ordinary expression typing), then inference and overload resolution stay the same as baseline Cangjie, except for ambiguities that are genuinely introduced by foreign types or interop APIs.

**Flow sensitivity:** Variables are always bound to a single native Cangjie type so no flow sensitivity hack is possible.

# Extern vs Any vs Dynamic

This section records how the proposed `Extern` type relates to Cangjie’s built-in `Any` and to a **hypothetical** `Dynamic` type modeled on C# `dynamic` (Cangjie does not provide such a type today).

## Comparison with `Any`

`Any` is as described in current Cangjie documentation: the empty interface `interface Any {}`, with all interfaces extending it and all non-interface types implementing it, so `T <: Any` for every type `T`. Values of type `Any` are ordinary Cangjie values; recovering a concrete type uses `as`, which yields an `Option`.


|                                 | **Any**                                                                                                                                              | **Extern** (this proposal)                                                                                                                                 |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Primary role**                | Universal supertype for heterogeneous *native* Cangjie values                                                                                        | Marker for values that *originate in foreign code*; opaque until converted to a native type                                                                |
| **Typical use**                 | Containers or APIs that intentionally erase to the top of the Cangjie lattice                                                                        | FFI parameters, results, and function types mentioning `Extern`; consume via boundary conversion to native **data** or explicit `as` / interop APIs        |
| **Operations before narrowing** | No extra “envelope” rules beyond normal typing; still need `as` or APIs to use as `Int32`, a class, etc.                                             | Must not eliminate `Extern` via guards, arithmetic, pattern matching, etc. until converted                                                                 |
| **Storage (this proposal)**     | Normal rules; `Any` is a standard interface type                                                                                                     | Locals and fields use *native* types only; no binding annotated or inferred as bare `Extern`, etc.; assignment targets must be native                      |
| **Type inference**              | Part of the baseline subtype hierarchy                                                                                                               | Intended not to subsume or break inference the way C# `dynamic` can                                                                                        |
| **Interop tooling**             | For example, TypeScript `any` is mapped to Cangjie `Any` in dts2cj-style generated code, with bridge helpers such as `Any.fromJSValue` / `toJSValue` | Intended to replace verbose “everything is `JSValue` + `asFunction()` / `asNumber()` …” patterns with surface types that still fail at explicit boundaries |


In short: **Any** is *maximal polymorphism* inside the Cangjie type system; **Extern** is *provenance and enforced boundary conversion* for foreign values.

## Comparison with a C#-style `Dynamic` (hypothetical)

Cangjie has no `Dynamic` type. For comparison, imagine a type **Dynamic** designed like C# **dynamic**: a static type that **defers** member lookup, operators, and conversions to **runtime**, via compiler-generated binding (as in C#’s runtime binder), so that expressions involving `Dynamic` are type-checked only to the extent the binder can succeed at run time.


|                                | **Hypothetical Dynamic (C#-style)**                                                                                                               | **Extern** (this proposal)                                                                                                                                      |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Intent**                     | Erasure of static typing in a **region** of code: call arbitrary “members,” apply operators, and pass values through APIs as if dynamically typed | Mark **foreign-origin** values and forbid treating them as native until an **explicit** extern-to-native conversion                                             |
| **Member access / operations** | Typically **allowed** at compile time; failures are **runtime** (e.g. missing member, bad argument types)                                         | **Disallowed** on bare `Extern` until converted; no open-ended “try any operation” surface                                                                      |
| **Provenance**                 | Not tied to FFI: any value could be lifted to `Dynamic`                                                                                           | Specifically **boundary** values from foreign code                                                                                                              |
| **Inference and checking**     | **Suppresses** normal static rules in `Dynamic`-typed subexpressions; can let ill-typed-looking programs compile and fail later                   | **Non-camouflage** and **inference conservativity**: cannot “fix” a program by widening to `Extern`; native-only code keeps baseline inference                  |
| **Failure locus**              | Wherever the runtime binder executes an operation                                                                                                 | Declared **boundary conversions** (casts, bindings to native **data**, **call sites** where a parameter or result is `Extern` or a function type mentioning it) |


**Can Dynamic replace Extern?** Not as specified here. A C#-style `Dynamic` is a **different mechanism**: late binding everywhere the type flows, not a sealed envelope with **only** conversion sites as checks. Conversely, **Extern** does not replace **Dynamic**: it does not offer arbitrary late-bound member calls on native or foreign values inside Cangjie code.

## Can `Any` replace `Extern` (or the reverse)?

**Not in general—they are complementary.**

1. **Extern** does not subsume **Any**. Use cases that need the universal supertype for *native* values (heterogeneous collections, APIs that erase to the top type, generated mappings from TS `any`) are modeled by `Any`. `Extern` does not mean “any Cangjie type”; it means “value from another runtime until decoded to native.” Replacing `Any` with `Extern` would mis-model purely native code and would clash with the placement rules in this document (e.g. forbidding `Extern` locals).
2. **Any** does not subsume **Extern**. Every native value is already an `Any`, so **Any** does not mark foreign provenance or enforce the “no elimination until native conversion” discipline described for `Extern`. For JS and similar runtimes, truly untyped foreign payloads are often represented today with **JSValue** (or analogous bridge types); this proposal’s `Extern` is a core-language boundary contract at that layer, not the same abstraction as `Any`.

**Summary:** `Any` answers “hold any Cangjie-typed value uniformly”; `Extern` answers “this crossed from foreign code—only treat it as native at explicit conversion sites.” Neither is a drop-in substitute for the other. A hypothetical C#-style `Dynamic` answers “defer static rules and bind at runtime”; that is again a different axis from both `Any` and `Extern`.

## Barring local `Extern` variables

1. Overload / target ambiguity (inference + Extern temporaries)

Suppose `api()` has type `Extern` and consider the following (the problematic case is *if* `let x = api()` were given type `Extern`—which this proposal disallows):

```cangjie
func process(n: Int32): Unit { ... }
func process(s: String): Unit { ... }
func bad(): Unit {
    let x = api()  // if allowed to have type Extern...
    process(x)    // which `process`? What conversion applies?
}
```

If `let x = api()` must not infer `Extern`, we reject this until the programmer writes something like `let x: Int32 = api()`. If `let x: Extern` (or inferred `Extern`) is allowed, the compiler either must add disambiguation rules or risk picking a default / widening path.

1. Same envelope, multiple native interpretations (subtle runtime failure)

This is perhaps the most problematic. The following is **not allowed** under this proposal; it is shown only to explain why bare `Extern` locals are forbidden:

```cangjie
func bad2(): Unit {
    let w: Extern = singleForeignValue()   // one foreign object
    let a: Int32 = w     // conversion succeeds
    let b: String = w    // later, treat *the same* `w` as String — may fail here
    ...
}
```

Nothing here requires illegal elimination on `w` before conversion: each line is a boundary. The problem is ergonomic / semantic: a hypothetical bare-`Extern` local encourages reusing one `Extern` value for several incompatible native roles. (This document forbids such a `let`; the snippet is illustrative of why.)

## Questions and answers

### How to interoperate with multiple languages

*Will one `Extern` be enough, or type family including `JSExtern`, `PyExtern` and others will be required to cover distinct interop cases?*

On `Extern` is enough as the conversion is delegated to external APIs provided for each interop scenario. For instance:

```cangjie
// declaring an external JavaScript API class
external class JSAPI { ...
  func Add(x: Int32, y: Int32) : Extern
...
}

// declaring an external Python API class
external class PyAPI { ...
  func Add(x: Int32, y: Int32) : Extern
...
}

// declaring interfaces to VMs
class JSVM {
  public func newAPI(module: String) : Extern
}

class PyVM {
  public func newAPI(module: String) : Extern
}

// interop with multiple languages
let jsvm = new JSVM()
let pyvm = new PyVM()

let jsapi: JSAPI = jsvm.newAPI("JSAPI")
let pyapi: PyAPI = pyvm.newAPI("PyAPI")

let result1: Int32 = jsapi.Add(2, 3)
let result2: Int32 = pyapi.Add(4, 5)
```

### How to have a stronger type discipline with ArkTS?

Suppose that we want to avoid functions such as 

```cangjie
foreign func formatDate(timestamp: Float64): Extern
```

and we would prefer using functions which are fully typed such as

```cangjie
foreign func formatDate(timestamp: Float64): String
```

This can be achieved as follows: 

```cangjie
// declaring an external JavaScript API class corresponding to a module "utils"
external class Utils { ...
  public extern func formatDate(timestamp: Float64): String
...
}

// declaring an interface to the JSVMs
class JSVM {
  public func newAPI(module: String) : Extern  // this is where Extern must be used
}
...
let jsvm = new JSVM()
let utils: Utils = jsvm.newAPI("Utils")        // retrieve an interface to "utils" and cast it into Utils
utils.formatDate(1.0)
```

### Comparison to other approaches

Another possibility is to use system-specific compiler instrumentation to maintain proxy objects between the Cangjie side and a designated special runtime, e.g. ArkTS. For instance:

```cangjie
@Interop[ArkTS, module: "geometry"]
foreign class Rectangle {
    var width: Float64
    var height: Float64
    init(w: Float64, h: Float64)
    func area(): Float64
}

func foo() {
    let rect = Rectangle(10.0, 20.0)
    rect.width = 30.0
    let a = rect.area()  // 200.0
}
```
In the proposed approach this would be as follows.

First we define an `external` class. All its members are handles to external data or functions. 
```cangjie
external class Rectangle {
    var width: Float64
    var height: Float64
    init(w: Float64, h: Float64)
    func area(): Float64
}
```
Instead of compiler instrumentation (`@Interop[ArkTS, module: "geometry"]`) we use an API to the target VM:

```cangjie
// A simple sample API to the external runtime
class VirtualMachine {
  // retrieve an object representing a module of that runtime using its name
  external public func import(module: String): Extern 
}

let vm = new VirtualMachine()
```

Note that the only function that *must* return an `External` type is in our example `import()` because it maps a module name into an object, which is also `external`. The members of the `external` class can be `External` but they can also be given designated Cangjie types, which means that they must always be converted to that type. 

The example user code is then exactly the same as above. 

The advantages our our approach:
* use language support rather than compiler instrumentation
* delegate the implementation of the interop to libraries
* support a uniform interface to any external language
* the `external` classes do not have to mirror the classes in the foreign langauge exactly, but the members can convert to-and-from data in the external counterpart. 
