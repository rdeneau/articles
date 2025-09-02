---
title: F# applicative computation expressions
description: Guide to write F# computation expressions having an applicative behavior
series: F# Computation Expressions
tags: #fsharp #dotnet #fp
cover_image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3lw7hnzaptmywhl7xta9.png
published: true
published_at: 2025-08-22 16:42 +0200
---

This fifth article in the series dedicated to F# computation expressions is a guide to writing F# computation expressions (CE) with [applicative](https://dev.to/rdeneau/functional-patterns-for-f-computation-expressions-46c7#applicative) behavior.

An applicative CE offers the widest variety of method combinations for the CE builder. We'll examine in detail when these are useful, based on compiler error messages and desugared versions of the expressions used in these tests.

{% collapsible Table of contents %}

* [Introduction](#introduction)
* [Builder method signatures](#builder-method-signatures)
* [CE Applicative example - `validation {}`](#ce-applicative-example-raw-validation-endraw-)
  * [Other method combinations](#other-method-combinations)
* [FsToolkit `validation {}`](#fstoolkit-raw-validation-endraw-)
  * [`Source` methods](#source-methods)
* [Conclusion](#conclusion)

{% endcollapsible %}

## Introduction

An applicative CE is revealed through the usage of the `and!` keyword _(F# 5)._

## Builder method signatures

An applicative CE builder should define these methods:

```fsharp
// Method        | Signature                        | Comment
    MergeSources : mx: M<X> * my: M<Y> -> M<X * Y>  ; (* ≡ *) map2 (fun x y -> x, y) mx my
    BindReturn   : m: M<T> * f: (T -> U) -> M<U>    ; (* ≡ *) map f m

// Additional methods for performance optimization
// - MergeSourcesN, N >= 3:
    MergeSources3: mx: M<X> * my: M<Y> * mz: M<Z> -> M<X * Y * Z>
// - BindNReturn, N >= 2:
    Bind2Return  : mx: M<X> * my: M<Y> * f: (X * Y -> U) -> M<U> ; (* ≡ *) map2 f mx my
// - BindN, N >= 2:
    Bind2        : mx: M<X> * my: M<Y> * f: (X * Y -> M<U>) -> M<U>
    Bind3        : mx: M<X> * my: M<Y> * mz: M<Z> * f: (X * Y * Z -> M<U>) -> M<U>
```

## CE Applicative example - `validation {}`

```fsharp
type Validation<'t, 'e> = Result<'t, 'e list>

type ValidationBuilder() =
    member _.BindReturn(x: Validation<'t, 'e>, f: 't -> 'u) =
        Result.map f x

    member _.MergeSources(x: Validation<'t, 'e>, y: Validation<'u, 'e>) =
        match (x, y) with
        | Ok v1,    Ok v2    -> Ok(v1, v2)     // Merge both values in a pair
        | Error e1, Error e2 -> Error(e1 @ e2) // Merge errors into a single list
        | Error e, _ | _, Error e -> Error e   // Short-circuit on single error source

let validation = ValidationBuilder()
```

**Usage:** validate a customer

* Name not null or empty
* Height strictly positive

```fsharp
type [<Measure>] cm
type Customer = { Name: string; Height: int<cm> }

let validateHeight height =
    if height <= 0<cm>
    then Error ["Height must be positive"]
    else Ok height

let validateName name =
    if System.String.IsNullOrWhiteSpace name
    then Error ["Name can't be empty"]
    else Ok name

module Customer =
    let tryCreate name height : Result<Customer, string list> =
        validation {
            let! validName = validateName name
            and! validHeight = validateHeight height
            return { Name = validName; Height = validHeight }
        }

let c1 = Customer.tryCreate "Bob" 180<cm>  // Ok { Name = "Bob"; Height = 180 }
let c2 = Customer.tryCreate "Bob" 0<cm> // Error ["Height must be positive"]
let c3 = Customer.tryCreate "" 0<cm>    // Error ["Name can't be empty"; "Height must be positive"]
```

Desugaring:

```fsharp
validation {                                ; validation.BindReturn(
                                            ;     validation.MergeSources(
    let! name = validateName "Bob"          ;         validateName "Bob",
    and! height = validateHeight 0<cm>      ;         validateHeight 0<cm>
                                            ;     ),
    return { Name = name; Height = height } ;     (fun (name, height) -> { Name = name; Height = height })
}                                           ; )
```

### Other method combinations

Let's try different combinations of methods to check what the compiler needs and what the desugaring looks like.

For convenience, let's start by defining some helpers:

* The `Err` active pattern simplifies pattern matching of `Validation<_, _>` tuples by limiting it to two cases: `ok` if all tuple elements are `Ok`, otherwise `error`. `Err` is used in `bind2` and `bind3`.
* `bind2` and `bind3` are the base helpers used to implement the builder methods that deal with two and three `Validation<_, _>` parameters respectively.

```fsharp
let (|Err|) (vx: Validation<'x, 'e>) =
    match vx with
    | Ok _ -> []
    | Error errors -> errors

module Validation =
    let ok (x: 'x) : Validation<'x, 'e> = Ok x
    let error (e: 'e) : Validation<'x, 'e> = Error [ e ]
    let errors (e: 'e list) : Validation<'x, 'e> = Error e

    let bind2 (vx: Validation<'x, 'e>) (vy: Validation<'y, 'e>) (f: 'x -> 'y -> Validation<'u, 'e>) : Validation<'u, 'e> =
        match vx, vy with
        | Ok x, Ok y -> f x y
        | Err ex, Err ey -> Error(ex @ ey)

    let bind3 (vx: Validation<'x, 'e>) (vy: Validation<'y, 'e>) (vz: Validation<'z, 'e>) (f: 'x -> 'y -> 'z -> Validation<'u, 'e>) : Validation<'u, 'e> =
        match vx, vy, vz with
        | Ok x, Ok y, Ok z -> f x y z
        | Err ex, Err ey, Err ez -> Error(ex @ ey @ ez)
```

#### Two-element expression

To handle an expression like `let! ... and! ...`, `Bind2` and `Return` form a valid combination for the compiler:

```fsharp
type ValidationBuilder() =
    member _.Bind2(vx: Validation<'x, 'e>, vy: Validation<'y, 'e>, f: 'x * 'y -> Validation<'u, 'e>) : Validation<'u, 'e> =
        Validation.bind2 vx vy (fun x y -> f (x, y))

    member _.Return(x: 't) =
        Validation.ok x

let test =
    validation {
        let! x = Validation.ok 1
        and! y = Validation.ok 10
        return x + y
    }

let test_desugared =
    validation.Bind2(
        Validation.ok 1,
        Validation.ok 10,
        (fun (x, y) -> validation.Return(x + y))
    )
```

When we look at the desugared version, we see that this combination of methods isn't really more efficient than the classic `BindReturn` and `MergeSources` combination. Let's try using only `Bind2Return`!

```fsharp
type ValidationBuilder() =
    member _.Bind2Return(vx: Validation<'x, 'e>, vy: Validation<'y, 'e>, f: 'x * 'y -> 'u) : Validation<'u, 'e> =
        Validation.bind2 vx vy (fun x y -> Validation.ok (f (x, y)))

let test =
    validation {
        let! x = Validation.ok 1
        and! y = Validation.ok 10
        return x + y
    }

let test_desugared =
    validation.Bind2Return(
        Validation.ok 1,
        Validation.ok 10,
        (fun (x, y) -> x + y)
    )
```

In this case, `Bind2Return` can be the only method needed by the compiler. The desugared result confirms that it's the only method call, leading to a more efficient builder implementation for handling `let! ... and! ...` expressions.

#### Three-element expression

The regular `BindReturn` and `MergeSources` method combination can handle an expression like `let! ... and! ... and! ...`:

```fsharp
type ValidationBuilder() =
    member _.BindReturn(x: Validation<'t, 'e>, f: 't -> 'u) : Validation<'u, 'e> =
        Result.map f x

    member _.MergeSources(vx: Validation<'x, 'e>, vy: Validation<'y, 'e>) : Validation<'x * 'y, 'e> =
        Validation.bind2 vx vy (fun x y -> Validation.ok (x, y))

let test =
    validation {
        let! x = Validation.ok 1
        and! y = Validation.ok 2
        and! z = Validation.ok 3
        return (z - x) * y
    }

let test_desugared =
    validation.BindReturn(
        validation.MergeSources(
            Validation.ok 1,
            validation.MergeSources(
                Validation.ok 2,
                Validation.ok 3
            )
        ),
        (fun (x, (y, z)) -> (z - x) * y)
    )
```

The desugared version shows that two `MergeSources` calls are needed to assemble the three values `x`, `y` and `z` required for the final calculation `(z - x) * y`. If the `MergeSources3` method is provided, the compiler can replace this double call to `MergeSources` with a single call to `MergeSources3`, simplifying the parameter deconstruction to obtain `x`, `y` and `z`—from `(x, (y, z))` to `(x, y, z)`:

```fsharp
type ValidationBuilder() =
    member _.BindReturn(x: Validation<'t, 'e>, f: 't -> 'u) : Validation<'u, 'e> =
        Result.map f x

    member _.MergeSources(vx: Validation<'x, 'e>, vy: Validation<'y, 'e>) : Validation<'x * 'y, 'e> =
        Validation.bind2 vx vy (fun x y -> Validation.ok (x, y))

    member _.MergeSources3(vx: Validation<'x, 'e>, vy: Validation<'y, 'e>, vz: Validation<'z, 'e>) : Validation<'x * 'y * 'z, 'e> =
        Validation.bind3 vx vy vz (fun x y z -> Validation.ok (x, y, z))

let test =
    validation {
        let! x = Validation.ok 1
        and! y = Validation.ok 2
        and! z = Validation.ok 3
        return (z - x) * y
    }

let test_desugared =
    validation.BindReturn(
        validation.MergeSources3(
            Validation.ok 1,
            Validation.ok 2,
            Validation.ok 3
        ),
        (fun (x, y, z) -> (z - x) * y)
    )
```

Note that `MergeSources` is not used, but is still required for compilation 🤷

Finally, we can verify that the locally optimal builder only needs a single method, `Bind3Return`:

```fsharp
type ValidationBuilder() =
    member _.Bind3Return(vx: Validation<'x, 'e>, vy: Validation<'y, 'e>, vz: Validation<'z, 'e>, f: 'x * 'y * 'z -> 'u) : Validation<'u, 'e> =
        Validation.bind3 vx vy vz (fun x y z -> Validation.ok (f (x, y, z)))

let test =
    validation {
        let! x = Validation.ok 1
        and! y = Validation.ok 2
        and! z = Validation.ok 3
        return (z - x) * y
    }

let test_desugared =
    validation.Bind3Return(
        Validation.ok 1,
        Validation.ok 2,
        Validation.ok 3,
        (fun (x, y, z) -> (z - x) * y)
    )
```

#### Method combination conclusion

The regular `BindReturn` and `MergeSources` method combination can handle all cases, but requires one additional call to `MergeSources` per additional input value. It's possible to optimize the number of builder method calls by providing additional builder methods. `BindNReturn` methods are the most efficient. `MergeSourcesN` methods are less efficient but more versatile as they can be combined to handle tuples with more elements.

## FsToolkit `validation {}`

[FsToolkit.ErrorHandling](https://github.com/demystifyfp/FsToolkit.ErrorHandling/) offers a similar `validation {}` implementation.

The desugaring reveals the definition of additional methods: `Delay`, `Run`, `Source`📍

```fsharp
validation {                                ;  validation.Run(
    let! name = validateName "Bob"          ;      validation.Delay(fun () ->
    and! height = validateHeight 0<cm>      ;          validation.BindReturn(
    return { Name = name; Height = height } ;              validation.MergeSources(
}                                           ;                  validation.Source(validateName "Bob"),
                                            ;                  validation.Source(validateHeight 0<cm>)
                                            ;              ),
                                            ;              (fun (name, height) -> { Name = name; Height = height })
                                            ;          )
                                            ;      )
                                            ;  )
```

### `Source` methods

In FsToolkit's `validation {}`, there are a couple of `Source` methods defined:

* The main definition is the `id` function.
* Another overload is interesting: it converts a `Result<'a, 'e>` into a `Validation<'a, 'e>`. Since it's defined as an extension method, it has lower priority for the compiler, leading to better type inference. Otherwise, we would need to add type annotations.

☝️ **Note:** `Source` documentation is scarce. The most valuable information comes from a [question on Stack Overflow](https://stackoverflow.com/a/35301315/8634147) referenced in the FsToolkit source code!

## Conclusion

Applicative computation expressions in F# enable parallel computation and error accumulation through the `and!` syntax introduced in F# 5. By implementing `MergeSources` and `BindReturn` methods, you can create powerful validation workflows that collect all errors rather than stopping at the first failure, as well as performant computations that leverage parallelism. This approach is particularly valuable for form validation, configuration parsing, and any scenario where you want to provide comprehensive feedback to users about multiple validation failures simultaneously. While applicative CEs are less versatile than monadic ones, they excel in specific use cases where their distinct capabilities make a significant difference.
