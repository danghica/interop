# Cangjie `Extern` type

# Lexical 
 * New keyword: `Extern`

# Types
* New type: `Extern`

## Extern 
`Extern` is a special type given to values which are not produced by a Cangjie expression, but are created by code written in another programming language. This can happen either when a foreign function is called by a Cangjie program, case in which it is its return type, or when a Cangjie function is called by a foreign context, case in which it is an argument type. Cangjie programs can propagate extern values but cannot read such values. In order to read them the values need to be first cast to Cangjie type. This cast may lead to a runtime error. 

Any Cangjie type that has no `Extern` component is said to be *native*. Any Cangjie type that has an `Extern` component is said to be *non-native*. For example the following types are non-native: `(Bool, Extern)` , `Array<Extern>`, `Option<Array<Extern>>`.

It is illegal to create any variables (mutable or immutable) of non-native type. The following example with generate compile-time errors: 

```[cangjie]
let x: Extern = e            // illegal even if expression e has Extern type
var y: Extern                // illegal even if y is left uninitialized
let z: (Extern, Int32) = e   // illegal because the variable type is non-native
let w = e                    // illegal because the type of w is inferred as Extern
```

However, if an expression has `Extern` type it can be cast into a mutable or immutable variable. It can also be assigned to a mutable variable. If `e1`, `e2` are expressions of type `Extern` the following are legal: 

```[cangjie]
let x: Int32 = e1
var y: Int32 = e1
y = e2
let z = (Bool)e1
```

Although legal, these expressions may produce a runtime error if the actual raw external data cannot be converted to the stipulate Cangjie type. 

If a non-native type and a native type differ only in that some data types are replaced with `Extern` the non-native type is said to be *extern-compatible* to the native type. For instance `(Bool, Extern)` is extern-compatible to `(Bool, Int32)`, `(Bool, Bool)` or `(Bool, (Bool, Int32))`. The type `Extern` itself is extern-compatible to any native type.    

The illegal and legal examples above generalize to types that are not extern-compatible and types which are extern-compatible, respectively. Suppose that `e` is an expressions
 that has type `(Extern, Bool)`. 

The following expressions give a compile error because they have types which are not extern-compatible to `(Extern, Bool)`: 
```[cangjie]
let x: Int32 = e           // illegal 
var y: (Bool, Int32) = e   // illegal 
var z: (Bool, Bool, Bool) 
z = e                      // illegal 
```

The following expressions are correct: 
```[cangjie]
let x: (Int32, Bool) = e           
var y: ((Bool, Bool), Bool) = e   
var z: (Bool, Bool) 
z = e                      
```


> **Optional**
> In general, whenever a Cangjie expression forces the type of a subexpression to be T and T' is an extern-compatible non-native type, an expression E of type T' can be used as such a subexpression. For instance the operator `&&` requires both operands to be `Bool`, therefore if `e1` and `e2` have type `Bool` or `Extern` then the expression `e1 && e2` is legal. 
> If an expression does not unambiguously determine the type of a subexpression then a cast is required. For instance, if `e1` , `e2` are `Extern` then `e1 + e2` will produce a compile-time error, whereas `(Int32)e1 + (Int32)e2` will be acceptable. 
> Here the concept of 'unambiguous' is to be strictly construed, i.e. no disambiguation mechanisms are required to uniquely determine the type. 
> This may require strengthening Cangjie type inference so it may be unacceptable. 

Redundant typecasts are allowed. For instance, in the example above `(Bool)e1 && (Bool)e2` is allowed. 

**Note:** To be added to functions and to generics that `Extern` can be used, e.g. in the identity function or as an instance of generics if no further problems occure. Check the feasability of runtime checks for boxed generics. Also think about whether function types are native or not. 

# Design rationale

The language design above will replace the following style of interop with ArkTS:

```[cangjie]
\\ call a function called 'Add(2, 3)' from ArkTS
let addF = api["Add"].asFunction()
let args: Array<JSValue> = [runtime.number(2).JsValue(), 
                            runtime.number(3).JsValue(), 
                            ]
let receiver:JSValue = runtime.null().toJSValue
let result = addF.call(args, thisArg: receiver)
                 .asNumber().toInt32()
```

To this new style:

```[cangjie]
\\ the type of api.Add() : (Int32, Int32) -> Extern
let result: Int32 = api.Add(2, 3)
```

Unlike the usual `dynamic` type used by C# which supresses type inference with dangerous consequences, the Cangjie approach: 

* cannot camouflage a badly-typed program 
* cannot replace normal type inference 
* cannot be used as a hack for flow-sensitive typing

