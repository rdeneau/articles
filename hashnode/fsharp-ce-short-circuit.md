---
title: "A real-world F# computation expression: short-circuiting computations"
description: A case study of a small custom F# computation expression that models short-circuiting steps with domain-meaningful names
tags: #fsharp #dotnet #fp
cover_image:
published: false
---

This article is a case study to complement the [F# Computation Expressions series](https://dev.to/rdeneau/f-computation-expressions-4ge6): rather than theory, it walks through a small `computation {}` CE pulled from production code and highlights two things the series never demonstrates in a real-world context.

## The problem: nested guards

Consider an error-message *humanization* pipeline: given a raw error message and a partner identifier, it tries to turn a technical string into something a user can read. The pipeline has four steps, and any step can produce a definitive outcome and stop:

1. **No message** — nothing to humanize.
2. **Unknown partner** — no rules exist for this partner; leave the message as-is.
3. **Already user-friendly** — the partner sends readable messages; keep them unchanged.
4. **Apply rules** — try to match the message against known patterns; a rule either fires or nothing matched.

Written as nested `match` expressions, the function looks like this:

```fsharp
let humanizeWithOutcome (message: Message option) (groupId: GroupId) : HumanizationOutcome<Message option> =
    match message with                                                                   // Step 1️⃣
    | None ->
        { Result = None; Status = HumanizationStatus.NoMessage }
    | Some message ->
        match helpersByGroupId |> Map.tryFind groupId with                               // Step 2️⃣
        | None ->
            { Result = Some message; Status = HumanizationStatus.NotHumanized }
        | Some helper ->
            match helper.DetermineSpecificity message with                               // Step 3️⃣
            | Some MessageSpecificity.AlreadyUserFriendly ->
                { Result = Some message; Status = HumanizationStatus.AlreadyUserFriendly }
            | specificity ->
                let context = { Message = message; Specificity = specificity }
                match helper.DetermineRules context |> Rule.tryApplyFirst context with   // Step 4️⃣
                | RuleNotEffective ->
                    { Result = Some message; Status = HumanizationStatus.NotHumanized }
                | RuleApplied humanizedMessage ->
                    { Result = Some humanizedMessage; Status = HumanizationStatus.Humanized }
```

The logic is correct, but it creeps rightward with every guard: four steps produce four levels of indentation. You have to reach the innermost branch to read the final outcome, and adding a fifth step pushes everything one level deeper.

### Why can't we just use early `return`?

In C#, guard clauses keep indentation flat:

```csharp
if (message is null) return new HumanizationOutcome(null, NoMessage);
if (!helpers.TryGetValue(groupId, out var helper)) return new HumanizationOutcome(message, NotHumanized);
// ...
```

F# doesn't have early `return` — **everything is an expression**, and an expression must have a single value. A `match` is an expression: the `else` branch (the continuation) is not optional, so it must be nested inside the `Some`/`true` branch. Each guard adds one indentation level.

Two partial workarounds exist, but neither is satisfying here:

* **Not indenting the** `else` **branch** — you can write the continuation at the same level as the guard, but this breaks standard formatting and Fantomas will re-indent it anyway.
* `elif` **chains** — they avoid nesting for simple boolean conditions, but fall short here: each step needs to **destructure a value** (`Some message`, `Some helper`, …) and **bind it to a name** for use in subsequent steps. `elif` can't do that.

So the nesting is structural: it is the direct consequence of expression-based semantics combined with destructuring bindings that must stay in scope.

💡 **Note:** This shape — a linear sequence of "if we have a definitive answer, return it; otherwise carry on with the in-progress value" steps — is exactly what a **short-circuiting monad** is designed for. See the [monad article in the series](https://dev.to/rdeneau/f-monadic-computation-expressions-4n0i) for the underlying theory; this article focuses on the real-world implementation.

## The `Computation` type

```fsharp
/// <summary>
/// Model steps of a computation that may be short-circuited.
/// </summary>
/// <remarks>
/// Isomorphic to the <c>Result</c> type, but with meaningful names and
/// a dedicated <c>computation {}</c> CE that automatically unwraps the
/// returned value or throws an exception if the computation is meant to be continued.
/// </remarks>
type Computation<'inner, 'outcome> =
    /// The computation continues, carrying the in-progress value of type `'inner`
    | ContinueWith of 'inner
    /// The computation has reached a definitive outcome and short-circuits; no further steps run.
    | Return of 'outcome
```

Two cases:

| Case             | Meaning                                                         | What happens next               |
| ---------------- | --------------------------------------------------------------- | ------------------------------- |
| `ContinueWith x` | Still undecided; `x` is the in-progress value for the next step | The next step runs with `x`     |
| `Return outcome` | Definitive answer reached                                       | All remaining steps are skipped |

👉 **Key point:** the names are chosen for the **domain narrative**, not the data-structure role. `ContinueWith` reads "not done yet, keep going"; `Return` reads "we have our answer, stop here." Neither implies success or failure — both `Humanized` and `NotHumanized` are valid outcomes.

💡 **Note:** The names are a deliberate nod to C# keywords: `return` (early exit from a method) and `continue` (skip to the next iteration of a loop). `Return` exits the pipeline early with a final answer; `ContinueWith` passes the in-progress value into the next step.

The type has **two independent type parameters**: `'inner` for the in-progress value passed between steps, and `'outcome` for the final result. These can be entirely different types, which is the case here: `'inner` is the pipeline's intermediate state, while `'outcome` is `HumanizationOutcome<Message option>`.

## The builder

```fsharp
module Computation =
    /// Monadic bind: runs `f` only while still `ContinueWith`;
    /// a `Return` outcome is propagated unchanged.
    let bind f =
        function
        | ContinueWith x -> f x              // 👈 still in progress: run the next step
        | Return outcome -> Return outcome   // 👈 already decided: skip remaining steps (short-circuit ⚡)

type ComputationBuilder() =
    member _.Bind(m, f)   = Computation.bind f m
    member _.Return x     = Return x
    member _.ReturnFrom m = m
    member _.Zero()       = ContinueWith ()

    member _.Run(m) =
        match m with
        | Return outcome -> outcome  // 👈 finalize: unwrap the definitive answer
        | ContinueWith _ -> failwith "Computation cannot be finalized as it is meant to be continued"

let computation = ComputationBuilder()
```

### `Bind`

`Bind` implements the `let!` desugaring and is the workhorse of the CE. Given a computation `m` and a continuation function `f`:

* If `m = ContinueWith x`, the continuation `f x` runs — the next step receives the in-progress value.
* If `m = Return outcome`, `f` is **never called** and `Return outcome` propagates unchanged through all remaining `let!` steps.

This is what "short-circuiting" means: `Bind` acts as a gate at each step, and a `Return` closes the gate for everything that follows in the `computation { … }` block.

### `Return` and `ReturnFrom`

* `Return x` wraps `x` into `Return x`, marking the computation as concluded.
* `ReturnFrom m` passes `m` through unchanged — used with `return!` when the value is already a `Computation`.

### `Zero`

`Zero` is called for `do!` expressions and `if` without an `else` branch. It returns `ContinueWith ()` — the `()` is `unit`, because there is no in-progress value to pass to the next step. This is what makes `do!` possible: instead of writing `let () = expr` or `let _ = expr`, you can write `do! expr` and the CE handles the `unit` result transparently.

### `Run` — the distinctive bit

`Run` is called by the CE machinery once the entire `computation { … }` block has been evaluated. It has two outcomes:

* `Return outcome → outcome` — the normal path: unwrap and return the final result directly. The CE's return type is `'outcome`, not `Computation<_, 'outcome>`.
* `ContinueWith _ → failwith "…"` — a guard: if the pipeline somehow reached the end without ever calling `Return`, that is a programming error and it fails loudly.

⚠️ **Warning:** The `failwith` branch is a guard, not a realistic code path. It can only trigger if the `computation { … }` block ends in a `ContinueWith` without a concluding `return`/`return!`. The type system cannot prevent this — but the unit test suite, which covers all business scenarios, verifies that every path terminates in a `Return` before the code ships.

📍 **Compare with the series:** articles on monoidal and writing CEs also use `Run`, but there it serves the `Delayed<T>` pattern — it *executes a deferred thunk*. Notice that this builder has **no** `Delay` **member**: there is nothing to defer. `Run` here plays a different role entirely: *finalizing and unwrapping a concluded computation*. Same method name, different responsibility.

## The CE in action

With the type and builder in place, the pipeline rewrites as:

```fsharp
let humanizeWithOutcome (message: Message option) (groupId: GroupId) : HumanizationOutcome<Message option> =
    computation {
        let! message =                                                           // Step 1️⃣
            match message with
            | None     -> Return { Result = None; Status = HumanizationStatus.NoMessage }
            | Some msg -> ContinueWith msg

        let! helper =                                                            // Step 2️⃣
            match helpersByGroupId |> Map.tryFind groupId with
            | None        -> Return { Result = Some message; Status = HumanizationStatus.NotHumanized }
            | Some helper -> ContinueWith helper

        let! context =                                                           // Step 3️⃣
            match helper.DetermineSpecificity message with
            | Some MessageSpecificity.AlreadyUserFriendly ->
                Return { Result = Some message; Status = HumanizationStatus.AlreadyUserFriendly }
            | specificity ->
                ContinueWith { Message = message; Specificity = specificity }

        return!                                                                  // Step 4️⃣
            match helper.DetermineRules context |> Rule.tryApplyFirst context with
            | RuleNotEffective         -> Return { Result = Some message; Status = HumanizationStatus.NotHumanized }
            | RuleApplied humanizedMessage -> Return { Result = Some humanizedMessage; Status = HumanizationStatus.Humanized }
    }
```

Each `let!` binding is a self-contained guard:

* The right-hand side produces either `Return outcome` (definitive answer → short-circuit, everything below is skipped) or `ContinueWith value` (carry on, bind `value` to the name on the left).
* The final step uses `return!` because there is no next step to feed — the computation concludes either way, so there is no in-progress value to bind.

💡 **Note:** In each `match`, the `Return` branch is intentionally placed **first** — mirroring an early-return guard clause style. The compiler does not require this ordering, but it reinforces the reading: "here is the exit condition, and if not, we continue."

Compare with the nested-match version: the CE stays at a **constant indentation level** regardless of how many steps the pipeline has. Adding a fifth step inserts a new `let!` block; nothing else moves.

### Desugaring: the real builder calls

The flat-looking `computation { … }` block is syntactic sugar. The compiler rewrites it into **nested calls** to the builder methods — exactly as described in the [series](https://dev.to/rdeneau/f-computation-expressions-4ge6): each `let! var = expr` becomes `Bind(expr, fun var -> rest)`, `return!` becomes `ReturnFrom`, and the whole block is wrapped in `Run` (there is no `Delay` member, so no `Delay` wrapping). Desugared, `humanizeWithOutcome` is:

```fsharp
let humanizeWithOutcome (message: Message option) (groupId: GroupId) : HumanizationOutcome<Message option> =
    computation.Run(
        // let! message = …                                                        // Step 1️⃣
        computation.Bind(
            (match message with
             | None     -> Return { Result = None; Status = HumanizationStatus.NoMessage }
             | Some msg -> ContinueWith msg),
            fun message ->
                // let! helper = …                                                 // Step 2️⃣
                computation.Bind(
                    (match helpersByGroupId |> Map.tryFind groupId with
                     | None        -> Return { Result = Some message; Status = HumanizationStatus.NotHumanized }
                     | Some helper -> ContinueWith helper),
                    fun helper ->
                        // let! context = …                                        // Step 3️⃣
                        computation.Bind(
                            (match helper.DetermineSpecificity message with
                             | Some MessageSpecificity.AlreadyUserFriendly ->
                                 Return { Result = Some message; Status = HumanizationStatus.AlreadyUserFriendly }
                             | specificity ->
                                 ContinueWith { Message = message; Specificity = specificity }),
                            fun context ->
                                // return! …                                       // Step 4️⃣
                                computation.ReturnFrom(
                                    match helper.DetermineRules context |> Rule.tryApplyFirst context with
                                    | RuleNotEffective             -> Return { Result = Some message; Status = HumanizationStatus.NotHumanized }
                                    | RuleApplied humanizedMessage -> Return { Result = Some humanizedMessage; Status = HumanizationStatus.Humanized })))))
```

☝️ **Notes:**

* The nesting that the `computation { … }` block hid is now explicit: each `Bind` takes the step's right-hand side as its first argument and a **continuation** lambda as its second. The lambda's parameter is the name bound by `let!` (`message`, `helper`, `context`), and its body is "everything below that step until the `}`."
* This nested shape is precisely what the CE saves you from writing — and, mirroring the original guards, it grows one level deeper per step. The sugar trades it for a constant-indentation block.
* `Bind` short-circuits here, not by deferring work: if its first argument is `Return outcome`, the continuation lambda is **never invoked**, so the inner `match` expressions below it never run. A `Return` at Step 2️⃣, say, means Steps 3️⃣ and 4️⃣ are skipped entirely.
* The outer `computation.Run(…)` is the finalizer: it unwraps the concluding `Return outcome` into the bare `'outcome` that `humanizeWithOutcome` returns — which is why the desugared expression, and the CE, has type `HumanizationOutcome<Message option>` rather than `Computation<_, _>`.

💡 **Tip:** You don't have to derive this by hand. As shown in the [first article of the series](https://dev.to/rdeneau/f-computation-expressions-4ge6), you can recover the desugared form mechanically with `Unquote`'s `<@ … @>` quotations.

## `Computation` *vs* `Result` and railway-oriented programming

`Computation<'inner, 'outcome>` is structurally isomorphic to `Result<'inner, 'outcome>`:

| `Computation`    | `Result`  | Structural role                         |
| ---------------- | --------- | --------------------------------------- |
| `ContinueWith x` | `Ok x`    | Carry on; the bind continues with `x`   |
| `Return e`       | `Error e` | Short-circuit; `e` propagates unchanged |

But the **semantic polarity is inverted** relative to Railway-Oriented Programming (ROP):

|                       | ROP with `result {}`                    | This CE with `computation {}`                  |
| --------------------- | --------------------------------------- | ---------------------------------------------- |
| What flows through    | `Ok` — the success value                | `ContinueWith` — "not decided yet"             |
| What short-circuits   | `Error` — a failure                     | `Return` — *the answer*                        |
| Mental model          | "Unless something goes wrong, continue" | "Unless we have a definitive answer, continue" |
| Return type of the CE | `Result<'ok, 'err>`                     | `'outcome` (unwrapped by `Run`)                |

In ROP, short-circuiting signals *something failed*. Here, short-circuiting signals *we reached a conclusion*. The answer is the interesting branch, not an error.

### Why not use `result {}` from FsToolkit?

You *could* model this with `Result<'inner, 'outcome>` and `result {}`:

```fsharp
let humanizeWithOutcome message groupId : HumanizationOutcome<Message option> =
    result {
        let! msg = message |> Option.toResult { Result = None; Status = HumanizationStatus.NoMessage }
        // …
    }
    |> Result.defaultWith id  // 👈 extract the `Error` branch — which is our actual outcome
```

Three reasons the custom CE is cleaner here:

1. **Naming carries intent.** `ContinueWith`/`Return` read as the domain story. Using `Ok`/`Error` where `Error` means "we have our answer" actively misleads the reader.
2. `Run` **enforces the invariant.** A `result {}`\-based version returns `Result<_, _>` — the caller must `match` or `|> Result.defaultWith` to unwrap the outcome. The custom `Run` makes the CE return `'outcome` directly, with no wrapper to peel off.
3. **No implied success/failure hierarchy.** `Computation<'inner, 'outcome>` treats both type parameters symmetrically. `Humanized` and `NotHumanized` are equal citizens; neither is a "success" or an "error."

The trade-off: a custom type has no FsToolkit/FSharpPlus ecosystem (`traverse`, `sequence`, `mapError`, etc.). That is fine here because the pipeline is fully linear and never needs to aggregate or transform errors.

## When to build a CE like this

✅ **Good fit:**

* A **fixed linear pipeline** where each step either concludes or passes an in-progress value to the next.
* The **conclusion is the interesting branch** (not a failure), so `Error`/`Result` names would be semantically confusing.
* You want **domain-meaningful names** that read as the computation's narrative.
* You want `Run` to **enforce the invariant** that the pipeline always concludes — a loud fail-fast guard that prevents a silently-wrong value if a future step accidentally omits its `Return`.
* The computation is **unit-tested** with business scenarios covering all paths, so the `failwith` guard is a safety net rather than a liability.

❌ **Skip it when:**

* The branching is ad-hoc rather than a linear pipeline.
* You already use `Result` everywhere and the naming difference does not improve clarity.
* You need the surrounding ecosystem (traversals, applicative combining, error accumulation).
* A plain `Result<'inner, 'outcome>` + `result {}` + a descriptive type alias is already clear enough.

💡 **Note:** The entire implementation fits in 36 lines including doc comments and the module-level `bind` helper. If the naming improves readability and `Run` provides a useful invariant, the overhead is negligible.

## Conclusion

A custom `computation {}` CE can make a real production pipeline read as its domain narrative rather than a data-structure manipulation. The implementation here is minimal — one discriminated union, one builder — and it adds two things the standard `result {}` does not:

* **Domain-meaningful names** (`ContinueWith`/`Return`) that communicate the computation's intent rather than a structural success/failure polarity.
* **A finalizing** `Run` **method** that enforces the invariant that the pipeline must always reach a definitive answer, and unwraps that answer directly into the CE's return type.

For the mechanics behind `Bind`, `Return`, `Zero`, `Run`, and the wider theory of monads and computation expressions, see the [F# Computation Expressions series](https://dev.to/rdeneau/f-computation-expressions-4ge6) — this article assumes familiarity with those concepts and focuses on the real-world application.

## Updates

* **2026-06-18** — Added the [*Desugaring: the real builder calls*](#desugaring-the-real-builder-calls) section, showing the nested `Bind`/`ReturnFrom`/`Run` calls that `computation { … }` compiles to.
