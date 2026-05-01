Change log
* ✅ simpler motivating examples; JSON is not a motivation for interop but for `dynamic`
* ✅ use `foreign` for functions
  * obsoleted by the use of `Extern<T>`
* ✅ associate types with external runtimes to prevent `x = e` runtime errors when both are `Extern`
  * `Extern<T>` where `T` is the type of the VM
* ✅ remove the implementation hints or mark as asides
* ✅ where is `Extern` in the type hierarchy?
  * between Any and Nothing, separate
  * use explicit type casts to Cangjie?
    * problematic because of subtyping (breaks meaning), syntax (clumsy), slow (option impl)
* ✅ illustrate all rules with examples
* ✅ parsing e.id vs e.id(e, e...); (e.id)() vs e.(id()) 
    * v.e send the parse tree to the VM
  * e.id for id a mangled identifier (excape)
    * up to proxy provider to give a mapping
  * both points above: don't break parser
* ✅ no rules for creation (just examples)
* ✅'Rectangle' compare with current approaches
* ✅ language support for errors and exceptions 
  * Base Exception or Effect
  * Conversion or interpretation errors
* ✅ Relationship with the type `Any`
* ✅ Is the `T` in `Extern<T>` a singleton?
* Maybe consider finalizers, since they are mostly used for interop?

  
# The Cangjie `Extern` type

## Introduction

The proposed type `Extern<T>` is the type of references to values that reside in a *foreign memory space* relative to Cangjie. 
These references can be bound to variables and function arguments, or can be produced by expressions. 
By a "foreign memory space" we mean memory that is managed by a different runtime or virtual machine than Cangjie, for instance ArkTS, JavaScript, or Python. We call it a *foreign runtime*.
We refer to other Cangjie type, for contrast, as *internal* Cangjie types. 

When handling such references the following principles apply:

* Cangjie makes no assumptions about these values in terms of types (at language level) or memory layout (at compiler and runtime level).
* The type `T` in `Extern<T>` is used to distinguish between several runtimes and must always implement the interface `Runtime`. 
* `Extern<T>` values can be converted to and from internal-typed values by special libraries which are registered with the compiler. The failure of this conversion will raise an `ExternConversionException`. 
* `Extern<T>` values offer a special set of capabilities usually associated with dynamic typing. Errors involving dynamic code execution by the runtime `T` will raise an `ExternDynamicException`. 
   

## Quick introduction

`Extern<T>` is a special Cangjie type that is governed by some specific rules which will be discussed below. 
Before giving all the details we show a flavour of how it is used. 

For example the following function looks up geographic location using latitude and longitude and returns a geographical object.
The data is `Extern<ArkTS>` because it resides in the foreign memory space of an ArkTS virtual machine. 

```cangjie
func lookup(lat: Float32, long: Float32): Extern<ArkTS>
```

The returned geographical object can have arbitrary complexity. 
For instance it can be a simple point landmark such as the Space Needle in Seattle, informally represented as

```
  coordinates: [-122.3493, 47.6205]
  name: "Space Needle",
  city: "Seattle",
  category: "Landmark"
```

Or it can be something more complex, such as the oval contour of the Colosseum in Rome:

```
  name: "The Colosseum",
  amenity: "historic_site",
  material: "travertine",
  built_in: "80 AD"
  type: Polygon,
  coordinates: 
            [12.493, 41.890], [12.494, 41.891], [12.491, 41.891], 
            [12.490, 41.890], [12.491, 41.889], [12.493, 41.890],
            [12.492, 41.890], [12.492, 41.891], [12.491, 41.891], 
            [12.491, 41.890], [12.492, 41.890]
```

In some scenarios the application developer may want to use a fully featured library for accessing such data and mapping it into internal Cangjie types precisely. 
But in many scenarios the application developer may want to explore the returned object dynamically. 



### Simple scenario: precise typing

In this application scenario the developer knows that the location data is always a landmark, therefore it can be cast into an internal type. 

```cangjie
struct Landmark {
  let coordinates: (Float32, Float32)  
  let name: String
  let city: String
  let category: Category
}
```

where `Category` is some `enum` defined elsewhere. 

Then the following code uses the lookup function to retrieve a landmark `lm`:

```cangjie
let lm: Landmark = lookup(-122.3493, 47.6205)
println(lm.name) // will print "Space Needle"
```

The way the developer should understand this code is as follows.
The return type `Extern<ArkTS>` of the function `lookup` needs to be instantiated to the type of some elsewhere defined runtime (`ArkTS`) which will tell who is responsible for executing `lookup` and serializing and deserializing data into an internal Cangjie value. 
The local variable will therefore store a `Landmark` object extracted from the `Extern<ArkTS>` returned by the `lookup`. 

Also note that this process may fail, case in which a special exception, to be discussed later, will be raised. 



### Complex scenario: dynamic typing

In this scenario the developer may be unwilling to implement a comprehensively typed interface to use the foreign library. 
Assume the only thing that the developer wants is to extract the name of the returned geographical object. 
In this case the Cangjie code will use the dynamic features of `Extern<T>`: it is possible to treat an `Extern<T>` as an object and using the `.` member syntax access any presumed data or function members, which will always have the (return) type of `Extern<T>`, with the same `T`.

The following code retrieves a landmark name `ln`:

```cangjie
let ln: String = lookup(12.487, 41.893).name
println(ln) // will print "Colosseum"
```

In this case what happens is the following: 
As before, the function will be executed and data will be serialized and deserialized by the `ArkTS` runtime. 
Because the result of `lookup` is `Extern<ArktTS>` it will instruct the proxy to interpret `.name` as a field access, which will have the type also `Extern<ArkTS>`. 
Finally it will produce a Cangjie `String` type and bind it to the local `ln`. 


### Design philosophy and general rules

The type `Extern<T>` is inspired by that of `dynamic` in C#, except that it must be explicitly associated to an external runtime `T`.
Unlike `dynamic`, which suspends the normal type rules in order to facilitate interop, `Extern` is deliberately designed for interop with `T`, making the abuses or mistakes that are common for `dynamic` less likely. 

The rule for using `Extern<T>` is as follows (in the context of [Conversion Between Value Types](https://cangjie-lang.cn/en/docs?url=%2F0.53.13%2Fspec%2Fsource_en%2FChapter_02_Types.html)):

Whenever a bidirectional conversion is required between a type `External<T>` and another type `R` the conversion will be effected by the runtime `T` via a special mechanism, unless:
1. The type `External<T> <: R` or `R <: External<T>` case in which language subtyping rules prevail.
2. The type `R ≡ External<U>` and `T ≡/ U` case in which a type error is reported. 

> *Notes (not for the spec):*
> * Currently, Rule 1 above applies only when `R ≡ Nothing` or `R ≡ Any`, but future iterations could make the `External` type covariant.
> * The use of an explicit cast has been explicitly considered but it has three problems:
>   * *Syntactic:* It is exceedingly verbose (`(e as T).getOrThrow()`) on each occurrence of a conversion.
>   * *Semantic:* Casting only works along subtyping. When the runtime type `Extern<U>` of e is a subtype of T, the value of e as T is Some(e); otherwise, the value is None. However, `Extern<U>` is usually not a subtype of `T`.
>   * *Performance:* Creating option types is known to currently have a noticeable overhead while communicating with external runtimes (particularly ArkTS) is often performance-sensitive.

## `Extern` variables

Consider `extexp` an expression that has `Extern<R>` type and `cjexp` an expression that has an internal type `T`. 

### Using `let`

The instantiated rules for using `Extern` types and let-bound variables are as follows:

1. `let x = extexp` and `let x: Extern<R> = extexp` are correct and equivalent and will result in a variable `x` of type `Extern<R>` having the value produced by `extexp` and linked to the runtime `R`. 
2. `let x: T = extexp` is correct and will result in the runtime `R` converting the value it produces to the internal `T` type (`T ≢ Any`) and giving that value to `x`; if the conversion fails then `ExternConversionException` is thrown.
3. `let x: Any = extexp` is correct and will not require any type conversion.
4. `let x: Extern<R> = cjexp` is correct and will result in the runtime `R` converting the value of `cjexp` from the internal type `T` to the runtime for `R`; if the conversion fails then `ExternConversionException` is thrown. 
5. `let x: Extern<S> = extexp` is correct if and only if `R ≡ S`. This is because the two runtimes cannot be assumed to have the capability to exchange data directly; using an intermediate variable of an internal type is required:

    ```cangjie
    let x: Extern<Python> = lookup(12.2, 34.1).name // type error Python != ArkTS
    let s: String = lookup(12.2, 34.1).name         // converted by ArkTS
    let x: Extern<Python> = s                       // converted by Python
    ```


### Using `var`

The rules for using `Extern` types and var-bound variables are similar to those for `let`.

1. `var x = extexp` and `var x: Extern<R> = extexp` are correct and equivalent and will result in a mutable variable `x` of type `Extern<R>` initialized with the value produced by `extexp`, linked to runtime `R`. 
2. `var x: T = extexp` is correct and will result in the runtime `R` converting the value it produces to the internal `T` type and initializing `x` with that value; if the conversion fails then `ExternConversionException` is thrown.
3. `var A: Any = extexp` is correct and requires no conversion.
4. `var x: Extern<R> = clexp` is correct and will result in the runtime `R` converting the value of `cjexp` from the intenal type `T` to the runtime for `R`; if the conversion fails then `ExternConversionException` is thrown. 
5. `var x: Extern<S> = extexp` is correct if and only if `R ≡ S`. The two runtimes cannot be assumed to be capable of exchanging data directly. The compiler will issue a type error. The initialization must be mediated by an internal type. 

The same rules apply to initializing assignment, i.e. when there is some code between the variable declaration and the initial assignment for the above cases. 

```Cangjie
var x: Extern<R>
// some code
x = extexp // correct

var x: Extern<R>
// some code
x = cjexp  // correct

var x: Extern<S>
// some code
x = extexp  // correct if and only if R ≡ S
```

### Using assignment

The rules for assignment involving `Extern` involve the same implict converstions as before:

1. If `var x: Extern<R>` then `x = extexp` is correct and is a normal Cangjie assignment requiring no converstion. 
2. If `var x: T` then `x = extexp` is correct (`T ≢ Any`). The runtime of `R` will convert its result to `T`, the type of `x`, and assign it to `x`; if the conversion fails then `ExternConversionException` is thrown.
3. If `var x: Any` then `x = extexp` is correct and requires no conversion. 
4. If `var x: Extern<R>` then `x = cjexp` is correct. The runtime of `R` will convert the result of `cjexp` from `T` to whatever the foreign runtime needs; if the conversion fails then `ExternConversionException` is thrown. *Note:* `x` is a Cangjie variable hosted by the current runtime, so the assignment does not mean that the converted value is actually sent to the runtime `R`.
5. If `x: Extern<S>` then `x = extexp` is correct if and only if `R ≡ S`, case in which it is a normal assignment. Otherwise a type error is reported. 


## Function calls

### Internal functions

It is legal for function parameters and results to use `Extern` and for function types to mention `Extern`. 
For instance the identity function works on extern values:

```cangjie
func id<T>(x: Extern<T>): Extern<T> { return x }
```

It is also legal for a function to select between two `Extern<T>` values.

```cangjie
func sel<T>(x: Extern<T>, y: Extern<T>): Extern<T> { if (test()) { x } else { y } }
```

The same implicit conversions rules already discussed for variables apply; if the conversion fails then `ExternConversionException` is thrown. 
Therefore:
* it is legal to pass an `Extern<T>` value to a function expecting an internally-typed argument.
* it is legal to pass an internally-typed argument value to a function expecting an `Extern<T>` argument.
* it is legal to pass an `Extern<T>` value to a function as an `Extern<R>` if and only if `R` and `T` are the same.
* `Any` and `Nothing` are an exception to the above because normal subtyping applies. 


### External functions

A special type of `foreign` functions is not required in association with `Extern` types since the dynamic mechanism of `Extern` is sufficient for the purpose (discussed later). 
If `vm: Extern<T>` then `vm.f(e1, e2)` will call an external function at `vm` with arguments `e1, e2`, returning also `Extern<T>`. 
Unlike `foreign func` external functions obtained via the dynamic feature of `Extern` are not subject to the restiction of being called in an `unsafe` block and can be treated as first-class citizens. 

For instance, a function changes the colour of some `ArkUI` button object `b: Extern<ArkTS>` can be created as

```cangjie
let setbColor : (Color) -> Unit = { (c: Color) => b.setColor(c) }
```

where `Color` is some internal  type for storing colour information. 


## Typing considerations

The type `Extern<T>` is a subtype of `Any` and a supertype of `Nothing` but otherwise unrelated to the hierarchy of internal types. 
The type `Extern<T>` may participate in the formation of composite data types such as tuples, structs, classes, etc. 
The type `Extern<T>` can be used to instantiate a generic, for instance `Array<Extern<T>>`. 
The type `Extern<T>` is invariant in `T`. 

> *Note:*
> The type `Extern<T>` may be revised to be variant in `T` in the future. 

The safe casting works for `Extern<T>` as for any type and this cast must not be confused with the implicit conversion to and from internal types. 

For instance let `x: Any`:
```cangjie
let y = (x as Extern<T>).getOrThrow()
// will type y as Extern<T> and bind it to the value of x
// if and only if its type is actually Extern<T>
// otherwise this throws an exception

let z = y as Int32
// this is a type error as Extern<T> is never a subtype of Int32
}
```

In polymorphic functions the rules for `External` ensure that the following function behaves correctly whether `T` is internal or external:

```cangjie
func f<T>(x:T): Any {
  let y: Any = x
   return y
}
```

> *Note:* The implicit conversion between extern-typed values to intern-type values ensures that the semantics of `as` need not change.
> If `as` has a different behaviour for `Extern` then the following code would require revising the semantics of `as`:
> ```cangjie
> func f<U, V>(x: U, y: V) {
>  let z = x as V
>  // the behaviour of as is no longer polymorphic on the type: internal does test and copy, external does conversion
> }
> ```
> Note that the typing discipline for polymorphic functions is such that implicit converstions are not required. 


## Dynamic features

A value of type `Extern<T>` is dynamic in the sense that it will be evaluated by an external runtime which also bears the responsibility of converting the resulting value into an internally-typed value. 
The type of the member and the argument and return type of the method are always `Extern<T>` and they remain associated with the same runtime `T`. 

If `e` is a maximal expression of `Extern<T>` it will be evaluated by the runtime of `T`, which will receive the parse tree of `e` for evaluation. If the evaluation fails then `ExternDynamicException` is thrown. 

By *maximal* we mean that it is not a subexpression of a larger `Extern<T>` expression. 
The typing rules of Cangjie apply, taking the rules for `Extern` into consideration. 

*Note:* The mapping of Cangjie identifiers cannot be assumed to be perfectly mapped onto external runtime identifiers, as lexical conversions may be different. It is the responsibility of the externally provided runtime to manage and document this mapping. 

### Examples

Consider two dynamic values related to different runtimes `vmpy: Extern<Python>` and `vmjs: Extern<ArkTS>`. 

The following code is legal:

```cangjie
vmjs.add(3, 4) // the type is Extern<ArkTS>
```
The type of add is `(Extern<ArkTS>, Extern<ArkTS>) -> Extern<ArkTS>` and the arguments 3 and 4 type check via conversion from `Int32`.
The parse tree will be sent to the `ArkTS` runtime. 

The following code is illegal: 

```cangjie
vmjs.add(vmpy.x, vmpy.y) // type error
```
The arguments have type `Extern<Python>` which cannot be converted to `Extern<ArkTS>`, therefore producing a type error. 

The following code is legal: 

```cangjie
func f(z: Extern<Python>): Int32 { z }  

vmjs.add(f(vmpy.x), f(vmpy.y)) // the type is Extern<ArkTS>
```

The `Python` runtime will process the subexpressions `vmpy.x` and `vmpy.y`. 
The function `f` will convert `Extern<Python>` to `Int32`. 
The `ArkTS` runtime will process the subexpression `vmjs.add(...)`, which will also involve orchestrating the evaluation of the subexpressions for `Python` above. 

Note that if variables `x` and `y` do not exist in `vmpy` or if function `add` does not exist in `vmjs` then the `ExternDynamicException` will be thrown. 



## Creation of `Extern` values

There is no Cangjie constructor for `Extern` and there is no specific Cangjie language support for constructing `Extern` data but the developer can do it in several ways. 
`Extern<T>` values are produced by dynamic `Extern` expressions or by compiler intrinsics.



## Implementation and API

In order to support the type `External<T>` a type `T` must be implemented which will implement a new standard library interface `Runtime`.

We require the type `T` to be a singleton. 

> *Note:*
> This is for the sake of simplicity and may be revised in the future.
> It means that each *instance* of a foreign runtime requires its own Cangjie type.
> This can be overcome programatically, I think, by using dispatcher-style runtimes. 

```cangjie
public interface Runtime {
   public func eval(e: std.ast.expr): Extern<Runtime>
   public func serialize<T>(e: Extern<Runtime>): T
   public func deserialize<T>(e: T): Extern<Runtime>
   // maybe finalizer interface
}
```

The compiler will provide an intrinsic function which can 'seed' a runtime with its first `Extern` value. 

```cangjie
@intrinsic
func bindExtern<T>(x: T) : Extern<T>
```


## External classes (typical usage)

We estimate that library developers will prefer to wrap access to foreign runtimes into libraries that hide the use of `External<T>`. 
Below we show a typical if very simple example. 

Direct language support for external classes (i.e. types of objects residing in a foreign runtime) is not offered. 
Accessing such objects is done using the dynamic features of `Extern`. 
For extra support native Cangjie wrappers can be used, as shown below. 
This design pattern can be automated with simple macros due to its regular nature. For instance:

```cangjie
class Rectangle {
    // The external object 
    private var blob: Extern<ArkTS>

    // The vm managing the object
    private var vm: Extern<ArkTS>

    // The Cangjie constructor calls the external constructor
    // to initialize the foreign object
    public Rectangle(vm: Extern<ArkTS>) {
      this.vm = vm
      this.blob = vm.Rectangle() 
    }

    // The fields of the Cangjie objects are props
    // wrapping the fields of the external object
    public prop width: Float64 {
      get() { 
    let w :Float 64 = blob.width 
        return w
      }
      set(value) { 
        blob.width = value 
      }

    public prop height: Float64 {
      get() { 
    let h: Float 64 = blob.height 
        return h
      }
      set(value) { 
        blob.height = value 
      }

    // The functions of the Cangjie object are wrappers
    // for the external functions
    func area(): Float64 {
        let a: Float64 = blob.area()
        return a
    }
}


// Usage:
// Assume a simple sample API to the external runtime
// Assume Rectangle is in module 'geometry'
class VirtualMachine {
  public func importModule(module: String): Extern<ArkTS> 
}

// ----------------------------------------------
// user code

// first Create proxy to an external runtime
let vm = new VirtualMachine()
let r = Rectangle(vm.importModule("geometry"))
r.width = 3.0
r.height = 2.0
let a = r.area()
println(r.height)
```

TODO : COMPARE WITH REFLECTION AND WITH NEW PROPOSAL

