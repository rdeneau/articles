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

1. **No message** (`NoMessage`) — nothing to humanize.
2. **Unknown partner** (`PartnerNotSupported`) — no rules exist for this partner; leave the message as-is.
3. **Already user-friendly** (`AlreadyUserFriendly`) — the partner sends readable messages; keep them unchanged.
4. **Apply rules** — try to match the message against known patterns; a rule either fires (`Humanized`) or nothing matched (`NotHumanized`).

Each outcome is a case of a dedicated discriminated union:

```fsharp
type HumanizationOutcome =
    | NoMessage                                 // nothing to humanize
    | PartnerNotSupported                       // no rules exist for this partner
    | AlreadyUserFriendly                       // the partner already sends readable messages
    | NotHumanized of originalMessage: string   // no rule matched; original kept
    | Humanized of humanizedMessage: string     // a rule produced a readable message
```

Written as nested `match` expressions, the function looks like this:

```fsharp
let humanizeWithOutcome (optionalMessage: string option) (groupId: GroupId) : HumanizationOutcome =
    match optionalMessage with                                                                  // Step 1️⃣
    | None ->
        HumanizationOutcome.NoMessage
    | Some message ->
        match helpersByGroupId |> Map.tryFind groupId with                                      // Step 2️⃣
        | None ->
            HumanizationOutcome.PartnerNotSupported
        | Some helper ->
            match helper.DetermineSpecificity message with                                      // Step 3️⃣
            | Some MessageSpecificity.AlreadyUserFriendly ->
                HumanizationOutcome.AlreadyUserFriendly
            | specificity ->
                let context = { Context.Message = message; Specificity = specificity }
                match helper.DetermineRules context |> applyEachRuleUntilSuccess context with   // Step 4️⃣
                | MessageUnchanged ->
                    HumanizationOutcome.NotHumanized message
                | MessageHumanized humanizedMessage ->
                    HumanizationOutcome.Humanized humanizedMessage
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

💡 **One approach does exist:** a continuation-passing CE with `Combine`/`Zero`/`Delay` support achieves literal `if cond then return x` guard clauses with no `else` branch. It trades a simple data type for a higher-order function. See [*A third way: a continuation monad for literal early return*](#a-third-way-a-continuation-monad-for-literal-early-return) below.

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

The type has **two independent type parameters**: `'inner` for the in-progress value passed between steps, and `'outcome` for the final result. These can be entirely different types, which is the case here: `'inner` is the pipeline's intermediate state, while `'outcome` is `HumanizationOutcome`.

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
let humanizeWithOutcome (optionalMessage: string option) (groupId: GroupId) : HumanizationOutcome =
    computation {
        let! message =                                                           // Step 1️⃣
            match optionalMessage with
            | None         -> Return HumanizationOutcome.NoMessage
            | Some message -> ContinueWith message

        let! helper =                                                            // Step 2️⃣
            match helpersByGroupId |> Map.tryFind groupId with
            | None        -> Return HumanizationOutcome.PartnerNotSupported
            | Some helper -> ContinueWith helper

        let! context =                                                           // Step 3️⃣
            match helper.DetermineSpecificity message with
            | Some MessageSpecificity.AlreadyUserFriendly ->
                Return HumanizationOutcome.AlreadyUserFriendly
            | specificity ->
                ContinueWith { Context.Message = message; Specificity = specificity }

        return!                                                                  // Step 4️⃣
            match helper.DetermineRules context |> applyEachRuleUntilSuccess context with
            | MessageUnchanged                  -> Return (HumanizationOutcome.NotHumanized message)
            | MessageHumanized humanizedMessage -> Return (HumanizationOutcome.Humanized humanizedMessage)
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
let humanizeWithOutcome (optionalMessage: string option) (groupId: GroupId) : HumanizationOutcome =
    computation.Run(
        // let! message = …                                                        // Step 1️⃣
        computation.Bind(
            (match optionalMessage with
             | None         -> Return HumanizationOutcome.NoMessage
             | Some message -> ContinueWith message),
            fun message ->
                // let! helper = …                                                 // Step 2️⃣
                computation.Bind(
                    (match helpersByGroupId |> Map.tryFind groupId with
                     | None        -> Return HumanizationOutcome.PartnerNotSupported
                     | Some helper -> ContinueWith helper),
                    fun helper ->
                        // let! context = …                                        // Step 3️⃣
                        computation.Bind(
                            (match helper.DetermineSpecificity message with
                             | Some MessageSpecificity.AlreadyUserFriendly ->
                                 Return HumanizationOutcome.AlreadyUserFriendly
                             | specificity ->
                                 ContinueWith { Context.Message = message; Specificity = specificity }),
                            fun context ->
                                // return! …                                       // Step 4️⃣
                                computation.ReturnFrom(
                                    match helper.DetermineRules context |> applyEachRuleUntilSuccess context with
                                    | MessageUnchanged                  -> Return (HumanizationOutcome.NotHumanized message)
                                    | MessageHumanized humanizedMessage -> Return (HumanizationOutcome.Humanized humanizedMessage))))))
```

☝️ **Notes:**

* The nesting that the `computation { … }` block hid is now explicit: each `Bind` takes the step's right-hand side as its first argument and a **continuation** lambda as its second. The lambda's parameter is the name bound by `let!` (`message`, `helper`, `context`), and its body is "everything below that step until the `}`."
* This nested shape is precisely what the CE saves you from writing — and, mirroring the original guards, it grows one level deeper per step. The sugar trades it for a constant-indentation block.
* `Bind` short-circuits here, not by deferring work: if its first argument is `Return outcome`, the continuation lambda is **never invoked**, so the inner `match` expressions below it never run. A `Return` at Step 2️⃣, say, means Steps 3️⃣ and 4️⃣ are skipped entirely.
* The outer `computation.Run(…)` is the finalizer: it unwraps the concluding `Return outcome` into the bare `'outcome` that `humanizeWithOutcome` returns — which is why the desugared expression, and the CE, has type `HumanizationOutcome` rather than `Computation<_, _>`.

💡 **Tip:** You don't have to derive this by hand. As shown in the [first article of the series](https://dev.to/rdeneau/f-computation-expressions-4ge6), you can recover the desugared form mechanically with `Unquote`'s `<@ … @>` quotations.

## Adding a final step: computing quality after the last bind

The pipeline above always concludes at Step 4️⃣ via `return!` — there is nothing left to do once rules have run. Now suppose a new requirement arrives: report *how well* the humanization went. Specifically, if the message carried an error code but no rule targeted that code specifically (a generic fallback fired instead), that is *degraded* quality — a signal that a dedicated rule should be added.

Two new types capture this:

```fsharp
type HumanizationQuality =
    /// A rule specifically targeting the message matched (known ErrorCode, or content-based rule)
    | OptimalQuality
    /// Only a generic/fallback rule matched despite the message having an ErrorCode
    /// — a specific rule targeting that ErrorCode should be added
    | DegradedQuality
```

The `Humanized` outcome now carries the quality alongside the message:

```fsharp
type HumanizationOutcome =
    | NoMessage
    | PartnerNotSupported
    | AlreadyUserFriendly
    | NotHumanized of originalMessage: string
    | Humanized of humanizedMessage: string * HumanizationQuality  // ← quality added
```

To compute quality, the pipeline needs to reach *after* Step 4️⃣ succeeds — which means Step 4️⃣ must no longer be the terminal step. The technique: instead of concluding with `return!`, bind the result with `let!` and `ContinueWith`, compute quality with a plain `let`, and conclude with `return`:

```fsharp
let humanizeWithOutcome (optionalMessage: string option) (groupId: GroupId) : HumanizationOutcome =
    computation {
        let! message =                                                           // Step 1️⃣
            match optionalMessage with
            | None -> Return HumanizationOutcome.NoMessage
            | Some message -> ContinueWith message

        let! helper =                                                            // Step 2️⃣
            match helpersByGroupId |> Map.tryFind groupId with
            | None -> Return HumanizationOutcome.PartnerNotSupported
            | Some helper -> ContinueWith helper

        let! context =                                                           // Step 3️⃣
            match helper.DetermineSpecificity message with
            | Some MessageSpecificity.AlreadyUserFriendly -> Return HumanizationOutcome.AlreadyUserFriendly
            | specificity -> ContinueWith { Context.Message = message; Specificity = specificity }

        let! humanizedMessage =                                                  // Step 4️⃣
            match helper.DetermineRules context |> applyEachRuleUntilSuccess context with
            | MessageUnchanged -> Return(HumanizationOutcome.NotHumanized message)
            | MessageHumanized humanizedMessage -> ContinueWith humanizedMessage

        let humanizationQuality =                                                // Step 5️⃣ (no short-circuit)
            match context.ErrorCode with
            | Some code when not (helper.HasSpecificRuleFor code) -> DegradedQuality
            | _ -> OptimalQuality

        return HumanizationOutcome.Humanized(humanizedMessage, humanizationQuality)
    }
```

Two changes from the 4-step version:

**Step 4️⃣: `return!` → `let!`**

The step still short-circuits on failure — `MessageUnchanged` produces `Return (NotHumanized message)` and nothing below runs. The difference is the success branch: instead of `Return (Humanized humanizedMessage)`, it now uses `ContinueWith humanizedMessage`. This keeps the pipeline open: `humanizedMessage` is bound in scope, and the computation continues into Step 5️⃣, exactly as the earlier `let!` binds did.

**Step 5️⃣: plain `let`, concluded by `return`**

Quality computation requires no short-circuit — it always produces a value. It is a regular `let` binding, not `let!`. The final line is `return HumanizationOutcome.Humanized(humanizedMessage, humanizationQuality)`, which desugars to `computation.Return(…)`, then unwrapped by `Run`.

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
let humanizeWithOutcome optionalMessage groupId : HumanizationOutcome =
    result {
        let! message = optionalMessage |> Option.toResult HumanizationOutcome.NoMessage
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

## A third way: a continuation monad for literal early return

[Borar](https://bsky.app/profile/borar.bsky.social) shared his [gist](https://gist.github.com/Savelenko/5e3f4b670b4d89a689d8713f1a73325c) that offers exactly the "early return" the earlier section says F# doesn't have — using a **continuation-passing-style (CPS) monad**.

### The insight

In the gist, each computation step is not a plain value but a **function that receives "the rest of the pipeline"** as an argument:

```fsharp
// Gist's core type: a computation is a function from "the rest" to the final async result
type ContAsync<'r,'a> = ('a -> Async<'r>) -> Async<'r>
```

Short-circuiting then means **ignoring the rest**:

```fsharp
// `early`: ignore the continuation `_k`; just return the final answer immediately
let early (a: 'a) : ContAsync<'a, _> = fun _k -> async { return a }
```

`Combine`/`Delay` — members the DU builder deliberately omits — and `Zero` are what make `if cond then return! early x` legal F# with no `else` branch.

### A simplified, generic, synchronous version

The gist hardcodes `Async`. Generalizing: make `'final` polymorphic and drop the async coupling. Rename `early` → `Return` and the monadic lift → `ContinueWith`, aligning with the DU CE's vocabulary:

```fsharp
/// A computation in CPS style. Receives `next` ("the rest") and yields the concluding `'final`.
type Continuation<'final, 'a> = ('a -> 'final) -> 'final

/// Short-circuit: ignore the rest and conclude with `answer`. (cf. the DU's `Return`)
let Return (answer: 'final) : Continuation<'final, 'a> = fun _next -> answer

/// Carry on: hand `value` to the rest of the pipeline. (cf. the DU's `ContinueWith`)
let ContinueWith (value: 'a) : Continuation<'final, 'a> = fun next -> next value

type ContinuationBuilder() =
    member _.Bind(comp, f) : Continuation<'final, 'b> = fun next -> comp (fun a -> f a next)
    member _.Return(answer: 'final) : Continuation<'final, 'a> = fun _next -> answer   // short-circuits ⚡
    member _.ReturnFrom(comp) = comp
    member _.Zero() : Continuation<'final, unit> = fun next -> next ()                 // if-no-else: carry on
    member this.Combine(guard, rest) = this.Bind(guard, fun () -> rest)
    member _.Delay(f) : Continuation<'final, 'a> = fun next -> f () next
    member _.Run(comp: Continuation<'final, 'final>) : 'final = comp id                // finalize with identity

let continuation = ContinuationBuilder()
```

Key points:

* `builder.Return` **short-circuits** (`fun _next -> answer`) — the opposite of the gist, where `builder.Return` *continues* (`fun k -> k a`). This means the CE keyword `return x` is already the short-circuit; no separate `early` helper is needed.
* `Zero()` returns `fun next -> next ()` — when a guard condition is *false*, execution carries on with `()`. `Combine(guard, rest)` wires this up: if `guard` continues, `rest` runs; if `guard` short-circuits, `rest` is skipped.
* `Delay` defers the rest of the block into a closure so it is not evaluated when a guard fires.
* `Run comp = comp id` finalizes by passing the identity function as the last continuation — the same philosophy as the DU CE's `Run`, and with a guaranteed type-safe result: no `failwith` needed (the type `Continuation<'final,'final>` already encodes that the pipeline must conclude).

### Guard clauses with no `else`

The payoff: boolean guards read like C#.

```fsharp
type ProcessOutcome = Skipped of string | Forbidden of string | Processed

let processDingus lookUpDingus checkPolicy : ProcessOutcome =
    continuation {
        if not (lookUpDingus ()) then
            return Skipped "dingus not found"   // 👈 no else!

        if not (checkPolicy ()) then
            return Forbidden "no access"        // 👈 no else!

        return Processed
    }
```

This is **impossible with the DU `computation {}`** — that builder has the `Zero` but lacks the `Combine`, so the compiler rejects `if` without `else` inside it.

### Applying it to `humanizeWithOutcome`

Because every humanization step **destructures a value** (`Some message`, `Some helper`, …) rather than testing a boolean, the if-no-else feature does not apply. The `continuation {}` version is *textually identical* to the `computation {}` version — only the builder name changes.

### Recovering async

`'final` is fully generic, so async is not lost. Set `'final = Async<'r>` and add one `Source` overload (o extension method) to `let!` bare async computations inline:

```fsharp
member _.Source(asyncComp: Async<'a>) : Continuation<Async<'r>, 'a> =
    fun next -> async { let! a = asyncComp in return! next a }
```

`Run` then finalizes to `Async<'r>` — recovering the gist's `runContAsync`. The sync builder above is the general core; the async builder is one instantiation.

### The three builders at a glance

| Aspect                             | `computation {}` (DU)                    | `result {}` (FsToolkit)            | `continuation {}` (CPS)                    |
| ---------------------------------- | ---------------------------------------- | ---------------------------------- | ------------------------------------------ |
| Underlying type                    | 2-case DU `Computation<'inner,'outcome>` | `Result<'ok,'err>`                 | function `('a -> 'final) -> 'final`        |
| Builder lines (approx.)            | ~10                                      | none (library-provided)            | ~10                                        |
| Builder difficulty                 | first-order, transparent                 | —                                  | higher-order (CPS); harder to grok/debug   |
| "carry on" / "short-circuit"       | `ContinueWith` / `Return`                | `Ok` / `Error`                     | `ContinueWith` / `Return` (ignores `next`) |
| Short-circuit in CE                | `return!`/`return` of a DU value         | `Error x` match arm                | bare `return x`                            |
| `if guard then …` with **no else** | ✗                                        | ✗                                  | ✓ (the C#-est form)                        |
| Async / effects                    | ✗ (sync only)                            | needs `asyncResult {}`             | ✓ via `'final = Async<_>` + `Source`       |
| CE return type                     | `'outcome` directly (unwrapped by `Run`) | `Result<_,_>` (caller must unwrap) | `'final` directly (unwrapped by `Run`)     |
| Debuggability                      | easy (plain DU values)                   | easy                               | harder (closure chains, deeper stacks)     |

For a **fixed linear pipeline whose steps bind and destructure values**, the DU `computation {}` is the sweet spot — minimal, transparent, easy to step through. Reach for the **continuation monad** when you need literal boolean guard clauses with no `else`, or want to interleave effects (async) without committing to a fixed wrapper type — accepting a more abstract builder. Use `result {}` when already in the `Result` ecosystem and the inverted polarity (`Error` = "our answer") is acceptable.

## Conclusion

A custom `computation {}` CE can make a real production pipeline read as its domain narrative rather than a data-structure manipulation. The implementation here is minimal — one discriminated union, one builder — and it adds two things the standard `result {}` does not:

* **Domain-meaningful names** (`ContinueWith`/`Return`) that communicate the computation's intent rather than a structural success/failure polarity.
* **A finalizing** `Run` **method** that enforces the invariant that the pipeline must always reach a definitive answer, and unwraps that answer directly into the CE's return type.

A third CE shape — the continuation monad — goes one step further: it enables literal `if cond then return x` guard clauses with no `else` branch, at the cost of a higher-order builder that is harder to debug. The [three-way comparison table](#the-three-builders-at-a-glance) captures when to reach for each.

For the mechanics behind `Bind`, `Return`, `Zero`, `Run`, and the wider theory of monads and computation expressions, see the [F# Computation Expressions series](https://dev.to/rdeneau/f-computation-expressions-4ge6) — this article assumes familiarity with those concepts and focuses on the real-world application.

## Updates

### 2026-06-19

* Migrated the running example from a `{ Result; Status }` record to the `HumanizationOutcome` discriminated union, making invalid states unrepresentable: the record allowed contradictions such as `{ Result = Some msg; Status = NoMessage }`.
* Added the [*Adding a final step: computing quality after the last bind*](#adding-a-final-step-computing-quality-after-the-last-bind) section, showing how to turn a terminal `return!` step into a `let!`/`ContinueWith` bind followed by a plain `return`.
* Added the [*A third way: a continuation monad for literal early return*](#a-third-way-a-continuation-monad-for-literal-early-return) section (thanks to [Borar](https://bsky.app/profile/borar.bsky.social/post/3molx4ff6ys2l)): a simplified, generic, synchronous continuation-passing CE that supports `if guard then return x` clauses with no `else`, plus a three-way comparison of the `computation {}`, `result {}`, and `continuation {}` builders.

### 2026-06-18

* Added the [*Desugaring: the real builder calls*](#desugaring-the-real-builder-calls) section, showing the nested `Bind`/`ReturnFrom`/`Run` calls that `computation { … }` compiles to.
