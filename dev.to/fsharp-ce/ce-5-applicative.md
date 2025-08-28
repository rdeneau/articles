---
title: F# applicative computation expressions
description: Guide to write F# computation expressions having an applicative behavior
series: F# Computation Expressions
tags: #fsharp #dotnet #fp
cover_image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3lw7hnzaaptmywhl7xta9.png
published: true
published_at: 2025-08-22 16:42 +0200
---
zz
This fifth article in the series dedicated to F# computation expressions is a guide to writing F# computation expressions having an [applicative](https://dev.to/rdeneau/functional-patterns-for-f-computation-expressions-46c7#applicative) behavior.

{% collapsible Table of contents %}

* [Introduction](#introduction)
* [Builder method signatures](#builder-method-signatures)
* [CE Applicative example - `validation {}`](#ce-applicative-example-raw-validation-endraw-)
* [Trap](#trap)
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

// Additional methods to optimize performances
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
        | Error e1, Error e2 -> Error(e1 @ e2) // Merge errors in a single list
        | Error e, _ | _, Error e -> Error e   // Short-circuit single error source

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

Let's add additional methods to check the result on the method calls after desugaring.

### Bind2 and Return

If we try to add `Bind2`, `Return` is required:

```fsharp
type ValidationBuilder() =
    member _.BindReturn(x: Validation<'t, 'e>, f: 't -> 'u) =
        Result.map f x
    member _.Bind2(x1: Validation<'t1, 'e>, x2: Validation<'t2, 'e>, f: 't1 * 't2 -> Validation<'u, 'e>) : Validation<'u, 'e> =
        match x1, x2 with
        | Ok v1, Ok v2 -> f (v1, v2)
        | Error e1, Error e2 -> Error(e1 @ e2)
        | Error e, Ok _
        | Ok _, Error e -> Error e
    member _.Return(x: 't) =
        Ok x
    member m.MergeSources(x: Validation<'t, 'e>, y: Validation<'u, 'e>): Validation<'t * 'u, 'e> =
        m.Bind2(x, y, m.Return)

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

It still needs call 2 methods - initially: `MergeSources` and `BindReturn`, now: `Return` and `Bind2`, but as we defined `MergeSources` based on `Bind2` and `Return`, it is a tiny improvement in this case.

### Bind2Return

```fsharp
type ValidationBuilder() =
    member _.BindReturn(x: Validation<'t, 'e>, f: 't -> 'u) = Result.map f x
    member _.Bind2Return(x1: Validation<'t1, 'e>, x2: Validation<'t2, 'e>, f: 't1 * 't2 -> 'u) : Validation<'u, 'e> =
        match x1, x2 with
        | Ok v1, Ok v2 -> Ok(f (v1, v2))
        | Error e1, Error e2 -> Error(e1 @ e2)
        | Error e, Ok _
        | Ok _, Error e -> Error e
    member m.MergeSources(x: Validation<'t, 'e>, y: Validation<'u, 'e>) : Validation<'t * 'u, 'e> =
        m.Bind2Return(x, y, id)

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

It is better now: `Bind2Return` is the only method called. 👍

## MergeSources3

TODO: mx: M<X> * my: M<Y> * mz: M<Z> -> M<X * Y * Z>

## Trap

⚠️ The compiler accepts that we define `ValidationBuilder` without `BindReturn` but with `Bind` and `Return`. But in this case, we can lose the applicative behavior and it enables monadic CE bodies!

## FsToolkit `validation {}`

[FsToolkit.ErrorHandling](https://github.com/demystifyfp/FsToolkit.ErrorHandling/) offers a similar `validation {}`.

The desugaring reveals the definition of more methods: `Delay`, `Run`, `Source`📍

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

In FsToolkit `validation {}`, there are a couple of `Source` methods defined:

* The main definition is the `id` function.
* Another overload is interesting: it converts a `Result<'a, 'e>` into a `Validation<'a, 'e>`. As it's defined as an extension method, it has a lower priority for the compiler, leading to a better type inference. Otherwise, we would need to add type annotations.

☝️ **Note:** `Source` documentation is scarce. The most valuable information comes from a [question on Stack Overflow](https://stackoverflow.com/a/35301315/8634147) mentioned in FsToolkit source code!

## Conclusion

Applicative computation expressions in F# enable parallel computation and error accumulation through the `and!` syntax introduced in F# 5. By implementing `MergeSources` and `BindReturn` methods, you can create powerful validation workflows that collect all errors rather than stopping at the first failure, as well as performant computations that leverage parallelism. This approach is particularly valuable for form validation, configuration parsing, and any scenario where you want to provide comprehensive feedback to users about multiple validation failures simultaneously. While applicative CEs are less versatile than monadic ones, they excel in specific use cases where their distinct capabilities make a significant difference.
