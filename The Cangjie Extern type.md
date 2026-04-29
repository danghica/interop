Main feedback points
* ✅ simpler motivating examples; JSON is not a motivation for interop but for `dynamic`
* ✅ use `foreign` for functions
* associate types with external runtimes to prevent `x = e` runtime errors when both are `Extern`
  * `Extern<T>` where `T` is the type of the VM
* remove the implementation hints or mark as asides
* language support for errors and exceptions 
  * Base Exception or Effect
* where is `Extern` in the type hierarchy?
  * between Any and Nothing, separate
  * use explicit type casts to Cangjie?
    * problematic because of subtyping (breaks meaning), syntax (clumsy), slow (option impl)
* illustrate all rules with examples
* parsing e.id vs e.id(e, e...); (e.id)() vs e.(id()) 
  * v.e send the parse tree to the VM
* e.id for id a mangled identifier (excape)
  * up to proxy provider to give a mapping
* both points above: don't break parser
* no rules for creation (just examples)
* 'Rectangle' compare with current approaches

# The Cangjie `Extern` type



## Introduction

The proposed type `Extern<T>` is the type of values that reside in a *foreign memory space* relative to Cangjie. 
These values can be bound to variables and function arguments, or can be produced by expressions. 
By a "foreign memory space" we mean memory that is managed by a different runtime or virtual machine than Cangjie, for instance ArkTS, JavaScript, or Python. We call it a *foreign runtime*.
We refer to other Cangjie type, for contrast, as *internal* Cangjie types. 

When handling such values the following principles apply:

* Cangjie makes no assumptions about these values in terms of types (at language level) or memory layout (at compiler and runtime level). 
* `Extern<T>` values can be mapped into internal Cangjie values by special third-party libraries which are registered with the compiler
* The type `T` in `Extern<T>` is used to distinguish between several runtimes and must always implement the interface `ExternalRuntime`. 
* `Extern<T>` values offer a special set of dynamic-type capability. 

The use of the tag `foreign func` is extended to indicate functions that are supplied by a foreign runtime.
A function tagged as `foreign` need not be executed in an  `unsafe` block if it is provided by a managed safe runtime such as ArkTS or Python. 
   
   

## Quick introduction

`Extern<T>` is a new type that can be used for variables and functions similarly to other Cangjie types. 
There are some specific rules which will be discussed below. 

For example the following foreign function looks up geographic location using latitude and longitude and returns a geographical object.
The data is `Extern<T>` because it resides in a foreign memory space. 

```cangjie
foreign func lookup<T>(lat: Float32, long: Float32): Extern<T>
```

The geographical object can have arbitrary complexity. 
For instance it can be a simple point landmark such as the Space Needle in Seattle, informally represented as

```JSON
  coordinates: [-122.3493, 47.6205]
  name: "Space Needle",
  city: "Seattle",
  category: "Landmark"
```

Or it can be something more complex, such as the oval contour of the Colosseum in Rome:

```JSON
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

In this application scenario the developer knows that the location data is always a landmark, therefore it can be cast into an internal Cangjie type. 

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
let lm: Landmark = lookup<ArkTS>(-122.3493, 47.6205)
println(lm.name) // will print "Space Needle"
```

The way the developer should understand this code is as follows.
The return type `Extern<T>` of the function `lookup` needs to be instantiated to the type of some elsewhere defined runtime (`ArkTS`) which will tell who is responsible for executing `lookup` and serializing and deserializing data into an internal Cangjie value. 
The local variable will therefore store a `Landmark` object extracted from the `Extern<ArkTS>` returned by the `lookup<ArkTS>`. 

Also note that this process may fail, case in which a special exception, to be discussed later, will be raised. 



### Complex scenario: dynamic typing

In this scenario the developer may be unwilling to implement a comprehensively typed interface to use the foreign library. 
Assume the only thing that the developer wants is to extract the name of the returned geographical object. 
In this case the Cangjie code will use the dynamic features of `Extern<T>`: it is possible to treat an `Extern<T>` as an object and using the `.` member syntax access any presumed data or function members, which will always have the (return) type of `Extern<T>`, with the same `T`.

The following code retrieves a landmark name `ln`:

```cangjie
let ln: String = lookup<ArkTS>(12.487, 41.893).name
println(ln) // will print "Colosseum"
```

In this case what happens is the following: 
As before, the function will be executed and data will be serialized and deserialized by the `ArkTS` runtime. 
Because the result of `lookup<ArkTS>` is `Extern<ArktTS>` it will instruct the proxy to interpret `.names` as a field access, which will have the type also `Extern<ArkTS>`. 
Finally it will produce a Cangjie `String` type and bind it to the local `ln`. 



### Design philosophy

The design of `Extern<T>` is inspired by that of `dynamic` in C#. 
The difference is that the requirement to provide an external runtime `T` will make it difficult to use `Extern` as an escape hatch from the type system, avoiding the dark patterns that `dynamic` is sometimes used for. 
Note that *difficult* is not *impossible*: however, avoiding the type system is mainly a misguided convenience and not a deliberate exercise; by making it significantly less convenient than simply using the type system the motivation to pursue dark patterns disappears. 
The type of the variable also makes it clear that access to that variable involves (de)serialization and is potentially expensive. 


## `Extern` variables

Consider `extexp` an expression that has `Extern<R>` type and `cjexp` an expression that has an internal Cangjie type `T`. 



### Using `let`

The rules for using `Extern` types and let-bound variables are as follows:

1. `let x = extexp` and `let x: Extern<R> = extexp` are correct and equivalent and will result in a variable `x` of type `Extern<R>` having the value produced by `extexp` and linked to the runtime `R`. 
2. `let x: T = extexp` is correct and will result in the runtime `R` converting the value it produces to the Cangjie `T` type and giving that value to `x`.
3. `let x: Extern<R> = cjexp` is correct and will result in the runtime `R` converting the value of `cjexp` from the Cangjie type `T` to the runtime for `R`.
4. `let x: Extern<S> = extexp` is incorrect if `R` and `S` are not equal. This is because the two runtimes cannot be assumed to have the capability to exchange data directly; using an intermediate variable of an internal type is required:

    ```cangjie
    let x: Extern<Python> = lookup<ArkTS>(12.2, 34.1).name // type error
    let s: String = lookup<ArkTS>(12.2, 34.1).name         // converted by ArkTS
    let x: Extern<Python> = s                              // converted by Python
    ```

> *Note:* It is a special property of the type `Extern<T>` to be automatically converted to and from internal types wherever the the type of the context requires.
> The use of an explicit cast has been explicitly considered but it has three problems:
> *Syntactic:* It is exceedingly verbose (`(e as T).getOrThrow()`) on each occurrence of a conversion.
> *Semantic:* When the runtime type `Extern<U>` of e is a subtype of T, the value of e as T is Some(e); otherwise, the value is None. However, `Extern<U>` is not a subtype of `T`.
> *Performance:* Creating option types has a noticeable overhead while communicating with external runtimes (particularly ArkTS) is often performance-critical
   

### Using `var`

The rules for using `Extern` types and var-bound variables are similar to those for `let`.

1. `var x = extexp` and `var x: Extern = extexp` are correct and equivalent and will result in a mutable variable `x` of type `Extern` initialized with value produced by `extexp` and, importantly, access to the proxy managing the value of `extexp`. 
2. `var x: T = extexp` is correct and will result in the proxy associated with `extexp` converting the value it produces to the Cangjie `T` type and initializing `x` with that value. 
3. `var x: Extern = cjexp` is incorrect: a new mutable variable `x` is not associated with a proxy so it cannot process the result of type `T` of expression `cjexp`. The compiler will issue a compile error. 

The same rules apply to initializing assignment, i.e. when there is some code between the variable declaration and the initial assignment for the above cases. 
For instance, this is correct:

```Cangjie
var x: Extern
// some code
x = extexp
```

This will lead to a compile error:

```Cangjie
var x: Extern
// some code
x = cjexp
```

### Using assignment

The rules for assignment involving `Extern` are as follows.

1. If `x: T` then `x = extexp` is correct. The proxy associated with the result of `extexp` will convert its result to `T`, the type of `x` and assign it to `x`. 
2. If `x: Extern` is an *initialized* mutable variable then `x = cjexp` is correct. The proxy associated with `x` will convert the result of `cjexp` from `T` to whatever the foreign runtime needs. 
3. If `x: Extern` is an *uninitialized* mutable variable then `x = cjexp` is incorrect. There is no proxy associated with `x` to convert the result of `cjexp` from `T` to whatever the foreign runtime needs. A compiler error will be issued. **Note:** The case `x = extexp` is correct, as an initializing assignment discussed above. 
4. If `x: Extern` is an *initialized* mutable variable then `x = extexp` is incorrect. The proxies for `x` or `extexp` may or may not be the same. If they are the same then the proxy could instruct the foreign runtime to perform the assignment. If they are not the same there is no schema for converting between the two. 

> *Obs:* Since assignment of `Extern` to `Extern` may sometimes succeed, and since programs involving `Extern` are not type safe Cangjie can be either strict (compile error) or permissive (compiler warning). In the latter case a special exception marking mismatched proxies should be raised. Note that rules 2-3 become much more useful if a permissive approach to assignment is taken. 



## Function calls

It is legal for function parameters and results to use `Extern` and for function types to mention `Extern`. 
For instance the identity function works on extern values:

```cangjie
func id(x: Extern): Extern { return x }
```

It is also legal for a function to select between two `Extern` values.

```cangjie
func sel(x: Extern, y: Extern): Extern { if (test()) { x } else { y } }
```

Because of the reasons already discussed for let-bound variables:

* it is legal to pass an `Extern` value to a function as a Cangjie-typed argument.
* it is legal to pass an `Extern` value to a function as an `Extern`-typed argument. 
* it is illegal to pass a Cangjie-typed value as an `Extern` argument

### External functions

The tag `foreign func` indicates that a function is not performed by the current Cangjie runtime. 
Functions tagged `foreign func` are declaration only, without a function body. 
External functions must be registered with a proxy using a special compiler-specific mechanism. 

For example: 

```cangjie
foreign func lookup(lat: Float32, long: Float32): External
```

Because external function arguments are passed to a foreign runtime, it is unsafe for them to be `Extern`-typed in case their proxies are different.
The permissive or restrictive approach must be applied consistently here as well. 



## Compound data types

The type `Extern` may participate in the formation of composite data types such as tuples, structs, classes, etc. 
The type `Extern` can be used to instantiate a generic, for instance `Array<Extern>`. 

> Note: The latter feature requires a permissive approach to `Extern` in assignment, otherwise safe assignment for instance at `Array<T>` may be deemed unsafe by the type system when `T` is `Extern`. 



## Dynamic features

A value of type `Extern` is dynamic in the sense that it can be decorated with arbitrary member access and method access operations. 
The type of the member and the return type of the method are always `Extern` and they remain associated with the same proxy. 

If `e: Extern` is a variable or expression and `id` is a valid Cangjie identifier then `e.id: Extern` is legal. 
Behind the scenes the proxy associated with `e` will be sent the identifier `id` as a string and it will be up to the proxy to resolve it. 

Similarly, `e.id(e1, e2, ...): Extern` is also legal if the types of `e1`, `e2`, etc. are Cangjie types. 
The runtime of `e` will be sent the identifier `id` and instructed to call a function of that name on the runtime with arguments the value of `e1`, `e2`, etc. These values have known CJ types and the proxy must handle their conversion to the data the runtime requires. 
It is illegal (for the same reasons discussed in the assignment of `Extern` to `Extern`) for the arguments to be `Extern`, as they may be transferred from one runtime to another. 
The decision to be strict or permissive should be consistent to that made in the case of assignment. 



## Creation of `Extern` values

Values of type `Extern` can only be created by the proxies, presumably by interfacing with the foreign runtime in scenario-dependent ways. 
There is no Cangjie constructor for `Extern`. 

There is no specific Cangjie language support for constructing `Extern` data but the developer can do it in several ways. 

1. The easiest way is to call foreign functions which produce new `Extern` data, as shown in the preliminary example (`lookup` function). 
2. Use factory methods and pass the name of the type we want created as an argument. For example imagine a proxy that manages access to an ArkTS runtime. That proxy may have as part of its API a method `createObject(typeName: String): Extern` which creates a remote object of type `typeName` in that runtime and return it as `Extern`. For instance `createObject("JSRectangle")` will instruct the ArkTS runtime to create a `JSRectangle` object and return it as `Extern` to the Cangjie runtime. 
3. Use the dynamic features of `Extern` to pass constructors. For instance if `vm` is some `Extern` value then `vm.JSRectangle(1, 2)`, which is legal syntax under the dynamic features of `Extern`, may be able to identify `JSRectangle` as a constructor of an ArkTS object and issue the correct call to the runtime. 

In order to preserve the generality of interaction between Cangjie and other runtimes via `Extern` no specific support will be provided for external constructors. 



## External classes

Direct language support for external classes (i.e. types of objects residing in a foreign runtime) is not offered. 
Accessing such objects is done using the dynamic features of `Extern`. 
For extra support native Cangjie wrappers can be used, as shown below. 
This design pattern can be automated with simple macros due to its regular nature. For instance:

```cangjie
class Rectangle {
    // The external object 
    private var blob: Extern

    // The vm managing the object
    private var vm: Extern

    // The Cangjie constructor calls the external constructor
    // to initialize the foreign object
    public Rectangle(vm: Extern) {
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
  public func import(module: String): Extern 
}

// ----------------------------------------------
// user code

// first Create proxy to an external runtime
let vm = new VirtualMachine()
let r = Rectangle(vm.import("geometry"))
r.width = 3.0
r.height = 2.0
let a = r.area()
println(r.height)
```


