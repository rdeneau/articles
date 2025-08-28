---
title: Writing F# computation expressions
description: Guide to write F# computation expressions
series: F# Computation Expressions
tags: #fsharp #dotnet #fp
cover_image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hjmc9soggopbsnty40nq.png
published: true
published_at: 2025-08-22 17:20 +0200
---

This final article in the series dedicated to F# computation expressions completes what we have seen regarding writing F# computation expressions of any kind: [applicative](https://dev.to/rdeneau/functional-patterns-for-f-computation-expressions-46c7#applicative), [monadic](https://dev.to/rdeneau/functional-patterns-for-f-computation-expressions-46c7#monad), or [monoidal](https://dev.to/rdeneau/functional-patterns-for-f-computation-expressions-46c7#monoid).

{% collapsible Table of contents %}

* [Types](#types)
  * [`M<T>` wrapper type](#mt-wrapper-type)
  * [`Delayed<T>` type](#delayedt-type)
  * [`Internal<T>` type](#internalt-type)
* [Builder-less CE](#builder-less-ce)
  * [Example: `activity {}`](#example-raw-activity-endraw-)
* [Custom operations 🚀](#custom-operations)
* [Final words](#final-words)
  * [Benefits ✅](#benefits)
  * [Limits ⚠️](#limits)
  * [Guidelines 📃](#guidelines)
  * [Tips 💡](#tips)
* [🍔 Quiz](#quiz)
* [🔗 Additional resources](#additional-resources)

{% endcollapsible %}

## Types

The CE builder method definitions can involve not 2 but 3 types:

* The wrapper type `M<T>`
* The `Delayed<T>` type
* An `Internal<T>` type

☝️ **Note:** we continue to use the generic type notation `Xxx<T>` for these types for convenience, even though it's an approximation.

### `M<T>` wrapper type

Candidates for this type are either generic types or "container" types like `string` as it contains `char`s. In fact, any type itself can be used as the wrapper type for a CE, as it can be written as the `Identity<T>` type: `type Identity<'t> = 't`. This was the case for the `logger {}` CE we saw in the first article of the series.

### `Delayed<T>` type

`Delayed<T>` is the type returned by the `Delay` method. It is used when we want to delay the evaluation of an expression inside the CE's body.

The `Delay` input parameter already involves this deferred evaluation as its type is `unit -> M<T>`, the type of a thunk. Based on that, we have three possibilities:

1. Eager evaluation: `Delay` consists in executing the thunk: `member _.Delay f = f()`. In this case, `type Delayed<T> = M<T>`. It is the default implementation when `Delay` is not required and not specified.
2. Deferred evaluation with no additional type: `Delay` returns the thunk directly, without executing it. `Delay` is just the identity function: `member _.Delay f = f`. `type Delayed<T> = unit -> M<T>`.
3. Deferred evaluation with an additional type: `Delay` uses the thunk to build an instance of an additional type, usually just wrapping the thunk: `member _.Delay f = Delayed f`.

Once the `Delay` method is defined, some of the other methods of the builder must be adapted: `Run`, `Combine`, `While`, `TryWith`, `TryFinally` must take into account that their input parameter has the `Delayed<T>` type.

#### `Delayed<T>` type example: `eventually {}`

In this example, adapted from the [Microsoft documentation](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions), we define a union type `Eventually<'t>` used for both wrapper and delayed types:

```fsharp
type Eventually<'t> =
    | Done of 't
    | NotYetDone of (unit -> Eventually<'t>)

type EventuallyBuilder() =
    member _.Return x = Done x
    member _.ReturnFrom expr = expr
    member _.Zero() = Done()
    member _.Delay f = NotYetDone f

    member m.Bind(expr, f) =
        match expr with
        | Done x -> f x
        | NotYetDone work -> NotYetDone(fun () -> m.Bind(work (), f))

    member m.Combine(command, expr) = m.Bind(command, (fun () -> expr))

let eventually = EventuallyBuilder()
```

The output values are meant to be evaluated interactively, step by step:

```fsharp
let step = function
    | Done x -> Done x
    | NotYetDone func -> func ()

let delayPrintMessage i =
    NotYetDone(fun () -> printfn "Message %d" i; Done ())

let test = eventually {
    do! delayPrintMessage 1
    do! delayPrintMessage 2
    return 3 + 4
}

let step1 = test |> step   // val step1: Eventually<int> = NotYetDone <fun:Bind@14-1>
let step2 = step1 |> step  // Message 1 ↩ val step2: Eventually<int> = NotYetDone <fun:Bind@14-1>
let step3 = step2 |> step  // Message 2 ↩ val step3: Eventually<int> = Done 7
```

### `Internal<T>` type

`Return`, `ReturnFrom`, `Yield`, `YieldFrom`, `Zero` methods can return a type internal to the CE. The `Combine`, `Delay`, and `Run` methods are adapted to handle this type.

For instance, we can review our `list {}` CE ([link](https://dev.to/rdeneau/f-monoidal-computation-expressions-2l06#ce-monoidal-to-generate-a-collection)) to use a `seq` type internally, as it is done by the list comprehension:

```fsharp
type ListSeqBuilder() =
    member inline _.Zero() = Seq.empty
    member inline _.Yield(x) = Seq.singleton x
    member inline _.YieldFrom(xs) = Seq.ofList xs
    member inline _.Delay([<InlineIfLambda>] thunk) = Seq.delay thunk
    member inline _.Combine(xs, ys) = Seq.append xs ys
    member inline _.For(xs, [<InlineIfLambda>] f) = xs |> Seq.collect f
    member inline _.Run(xs) = xs |> Seq.toList

let listSeq = ListSeqBuilder()
```

💡 **Note:** the `Internal<T>` type highlights the usefulness of `ReturnFrom` and `YieldFrom`, implemented as an _identity_ function until now.

## Builder-less CE

Up to now, we've assumed that a specific type had to be defined to serve as a _Builder_ to create a computation expression. It turns out that this isn't necessary. In fact, any type, even an existing one, can be extended to support CE syntax: simply extend it using extension methods of a CE builder.

Let's look at an example: the `activity {}` CE. It was written by my teammate Loïc/Tarmil, creator and maintainer of [Bolero](https://github.com/fsbolero/Bolero).

The purpose of the `activity {}` CE is to configure an `Activity` (from `System.Diagnostics`) with a lightweight convenient syntax.

Given an `activity` provided by any `ActivitySource`, we would like to write something like that:

```fsharp
use activity = Activities.source.StartActivity(...)

activity {
    setStartTime DateTime.UtcNow
    setTag "count" 2
}
```

Some preliminary remarks:

* The type to extend to support CE syntax is `System.Diagnostics.Activity`.
* The returned type is `unit`: the CE is only performing a side effect to change/mutate the `activity`.
* The CE involves implicit `yield`s for each call to helper methods like `setStartTime`, defined aside the extension methods.
* The internal functioning of the CE is based on the type `type ActivityAction = delegate of Activity -> unit`.
* Each helper creates an instance of `ActivityAction` that defines the delayed change on the `activity`. E.g. `let inline setStartTime time = ActivityAction(fun ac -> ac.SetStartTime(time) |> ignore)`.
* Internally, the CE combines every yielded `ActivityAction` that is created by the helpers. So, it's a monoidal CE.
* Externally, the CE looks like a `State` monad, with a series of `Set`.

Here the full code listing:

```fsharp
type ActivityAction = delegate of Activity -> unit

// Helpers
let inline private action ([<InlineIfLambda>] f: Activity -> _) =
    ActivityAction(fun ac -> f ac |> ignore)

let inline addLink link = action _.AddLink(link)
let inline setTag name value = action _.SetTag(name, value)
let inline setStartTime time = action _.SetStartTime(time)

// CE Builder Methods
type ActivityExtensions =
    [<Extension; EditorBrowsable(EditorBrowsableState.Never)>]
    static member inline Zero(_: Activity | null) = ActivityAction(fun _ -> ())

    [<Extension; EditorBrowsable(EditorBrowsableState.Never)>]
    static member inline Yield(_: Activity | null, [<InlineIfLambda>] a: ActivityAction) = a

    [<Extension; EditorBrowsable(EditorBrowsableState.Never)>]
    static member inline Combine(_: Activity | null, [<InlineIfLambda>] a1: ActivityAction, [<InlineIfLambda>] a2: ActivityAction) =
        ActivityAction(fun ac -> a1.Invoke(ac); a2.Invoke(ac))

    [<Extension; EditorBrowsable(EditorBrowsableState.Never)>]
    static member inline Delay(_: Activity | null, [<InlineIfLambda>] f: unit -> ActivityAction) = f()

    [<Extension; EditorBrowsable(EditorBrowsableState.Never)>]
    static member inline Run(ac: Activity | null, [<InlineIfLambda>] f: ActivityAction) =
        match ac with
        | null -> ()
        | ac -> f.Invoke(ac)

// ---

let activity = new Activity("Tests")

activity {
    setStartTime DateTime.UtcNow
    setTag "count" 2
}

// Desugaring
let _desugar =
    ActivityExtensions.Run(activity,
        ActivityExtensions.Delay(activity, (fun () ->
            ActivityExtensions.Combine(activity,
                ActivityExtensions.Yield(activity, setStartTime DateTime.UtcNow),
                ActivityExtensions.Delay(activity, (fun () ->
                    ActivityExtensions.Yield(activity, setTag "count" 2)
                ))
            ))
        )
    )
```

☝️ **Notes:**

* The `Delay` method evaluates the thunk `f: unit -> ActivityAction` to return the wrapped `ActivityAction` already involving a deferred action.
* The `Combine` method is used to chain two `ActivityAction`s into one, calling each one in series.
* The final `Run` is the only method really using the input `activity`. It evaluates the built `ActivityAction`, resulting in the change/mutation of the `activity`.
* The extension methods are marked as not `EditorBrowsable` to improve the developer experience: when we use dot notation on the `activity`, the extension methods are not suggested for code completion.

## Custom operations 🚀

What: builder methods annotated with `[<CustomOperation("myOperation")>]`

Use cases: add new keywords, build a custom DSL. For example, the `query` core CE supports `where` and `select` keywords like LINQ.

⚠️ **Warning:** you may need additional things that are not well documented:

* Additional properties for the `CustomOperation` attribute:
  * `AllowIntoPattern`, `MaintainsVariableSpace`
  * `IsLikeJoin`, `IsLikeGroupJoin`, `JoinConditionWord`
  * `IsLikeZip`...
* Additional attributes on the method parameters, like `[<ProjectionParameter>]`

🔗 [Computation Expressions Workshop: 7 - Query Expressions | GitHub](https://github.com/panesofglass/computation-expressions-workshop/blob/master/exercises/07_Queries.pdf)

## Final words

Let's review the pros and cons of computation expressions to get the full picture and make the appropriate decision about writing our own computation expression.

### Benefits ✅

Computation expressions offer significant advantages for F# developers. They provide **increased readability** through imperative-like code that feels natural while maintaining functional principles. They also **reduce boilerplate** by hiding complex "machinery" behind clean, expressive syntax. Additionally, their **extensibility** allows developers to extend existing CEs or even add the CE syntax support to any type. Finally, we can create **domain-specific languages** (DSLs) to reify domain concepts through custom operations.

### Limits ⚠️

However, computation expressions come with certain limitations that developers should be aware of. **Compiler error messages** within CE bodies can often be cryptic and difficult to debug, making troubleshooting more challenging. **Nesting different CEs** can make code more cumbersome to work with—for example, combining `async` and `result` patterns. While custom combining CEs like `asyncResult` in [FsToolkit](https://demystifyfp.gitbook.io/fstoolkit-errorhandling/#a-motivating-example) offer alternatives, they add complexity. Finally, **writing custom CEs can be challenging**, requiring developers to implement the right methods correctly and understand the underlying functional programming concepts.

### Guidelines 📃

* Choose the main **behaviour**: monoidal? monadic? applicative?
  * Prefer a single behaviour unless it's a generic/multi-purpose CE
* Create a **builder** class
* Implement the main **methods** to get the selected behaviour
* Use/Test your CE to verify it compiles _(see typical compilation errors below)_, produces the expected result, and performs well.

```txt
1. This control construct may only be used if the computation expression builder defines a 'Delay' method
   => Just implement the missing method in the builder.
2. Type constraint mismatch. The type ''b seq' is not compatible with type ''a list'
   => Inspect the builder methods and track an inconsistency.
```

### Tips 💡

* Get inspired by existing codebases that provide CEs - examples:
  * FSharpPlus → `monad`
  * FsToolkit.ErrorHandling → `option`, `result`, `validation`
  * [Expecto](https://github.com/haf/expecto): Testing library (`test "..." {...}`)
  * [Farmer](https://github.com/compositionalit/farmer): Infra as code for Azure (`storageAccount {...}`)
  * [Saturn](https://saturnframework.org/): Web framework on top of ASP.NET Core (`application {...}`)
* Overload methods to support more use cases like different input types
  * `Async<Result<_,_>>` + `Async<_>` + `Result<_,_>`
  * `Option<_>` and `Nullable<_>`

## 🍔 Quiz

#### Question 1: **What is the primary purpose of computation expressions in F#?**

**A.** To replace all functional programming patterns

**B.** To provide imperative-like syntax for sequencing and combining computations

**C.** To eliminate the need for type annotations

**D.** To make F# code compatible with C#

{% details Answer %}

**B.** To provide imperative-like syntax for sequencing and combining computations ✅

{% enddetails %}

#### Question 2: **Which keywords identify a monadic computation expression?**

**A.** `yield` and `yield!`

**B.** `let!` and `return`

**C.** `let!` and `and!`

**D.** `do!` and `while`

{% details Answer %}

**A.** `yield` and `yield!` keywords identify a monoidal CE ❌

**B.** `let!` and `return` keywords identify a monadic CE ✅

**C.** `let!` and `and!` keywords identify a applicative CE ❌

**D.** `do!` and `while` keywords can be used with any kind of CE ❌

{% enddetails %}

#### Question 3: **In a computation expression builder, what does the `Bind` method correspond to?**

**A.** The `yield` keyword

**B.** The `return` keyword

**C.** The `let!` keyword

**D.** The `else` keyword when omitted

{% details Answer %}

**A.** The `yield` keyword corresponds to the `Yield` method ❌

**B.** The `return` keyword corresponds to the `Return` method ❌

**C.** The `let!` keyword corresponds to the `Bind` method ✅

**D.** The `else` keyword, when omitted, corresponds to the `Zero` method ❌

{% enddetails %}

#### Question 4: **What is the signature of a typical monadic `Bind` method?**

**A.** `M<T> -> M<T>`

**B.** `T -> M<T>`

**C.** `M<T> * (T -> M<U>) -> M<U>`

**D.** `M<T> * M<U> -> M<T * U>`

{% details Answer %}

**A.** `M<T> -> M<T>` is the typical signature of `ReturnFrom` and `YieldFrom` methods ❌

**B.** `T -> M<T>` is the typical signature of `Return` and `Yield` methods ❌

**C.** `M<T> * (T -> M<U>) -> M<U>` is the typical signature of the `Bind` method ✅

**D.** `M<T> * M<U> -> M<T * U>` is the typical signature of `MergeSources` method ❌

{% enddetails %}

## 🔗 Additional resources

* [Code examples in FSharpTraining.sln](https://github.com/rdeneau/formation-fsharp/tree/main/src/FSharpTraining) —Romain Deneau
* [_The "Computation Expressions" series_](https://fsharpforfunandprofit.com/series/computation-expressions/) —F# for Fun and Profit
* [All CE methods | Learn F#](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions#creating-a-new-type-of-computation-expression) —Microsoft
* [Computation Expressions Workshop](https://github.com/panesofglass/computation-expressions-workshop)
* [The F# Computation Expression Zoo](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/computation-zoo.pdf) —Tomas Petricek and Don Syme
  * [Documentation | Try Joinads](http://tryjoinads.org/docs/computations/home.html) —Tomas Petricek
* Extending F# through Computation Expressions: 📹 [Video](https://youtu.be/bYor0oBgvws) • 📜 [Article](https://panesofglass.github.io/computation-expressions/#/)
