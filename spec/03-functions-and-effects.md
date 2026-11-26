# 03 — Functions, Methods, Effects and Pipelines

**Status: DRAFT.**

A free function is declared with `fnc`; class and actor instance methods use an implicit `self` and canonically omit `fnc`. Every named callable must explicitly declare its return type, including `void`; explicit `-> infer` (or `: infer`) is allowed, but a concrete result type is preferred. Omitted return annotations are invalid. Each invocation returns **one value**; tuples/records are themselves one value.

## Mandatory named-callable return annotations

The accepted AttoLang v1 rule is that **every named function, routine, class/actor instance or static method, out-of-line method, and actor receive handler must explicitly annotate its return**. Accepted syntax includes name-first `name(params) -> T` or `name(params): T` and return-first `[fnc] T <- name(params)`, where `T` can be a concrete type, `void`, or the explicitly requested inference placeholder `infer`. The return-first form also allows braces, `begin`/`end`, and an explicit statement/`end` body; its `<-` token is not an assignment operator. Inference is permitted but less desirable for public APIs and cross-module signatures; the compiler resolves it statically or reports an error, never silently choosing void. For example:

~~~atto
fnc calculate(int x) -> int { return x + 1; }
fnc shutdown() -> void { }
fnc choose() -> infer { return 42; }
~~~

A constructor has no return annotation, since construction is not an ordinary return-valued callable; lambdas retain separately specified contextual inference. See the [normative return-type contract](https://github.com/atto-lang/atto-spec/blob/main/language/explicit-callable-return-types-v1.md). This is an accepted target; backend execution and formatter parity remain separately gated.

## Signatures and visibility

```atto
pub fnc square(int x) -> int ::
  return x * x
end

define class Ledger as
  let int total: 0

  add(int amount) -> void ::
    self.total = self.total + amount
  end

  static fnc zero() -> int ::
    return 0
  end
end
```

This `-> T :: ... end` style is one of the valid [function forms](16-lexical-and-grammar-reference.md#function-forms). Backend support differs until grammar parity is proved.

## Named function, method and constructor arguments (v1 target)

Runtime value arguments use **`label: expression`**, whereas generic type arguments use **`T=Type`**. Ordinary assignment in a call is **not** a named argument. This surface is intended for ordinary functions, instance/static methods and constructors with statically known parameter names; dynamic/foreign/opaque callables without a verified name-bearing signature reject named calls.

```atto
fnc combine(int first, int second) -> int ::
  return first * 10 + second
end

val a = combine(second: 2, first: 1) // 12
val b = combine(1, second: 2)         // 12
// obj.set_value(value: 7, mode: #checked)
// Pair(first: 1, second: 2) uses the same binder only if Pair exposes named constructor formals.
```

**Checked binding plan:** Resolve the callable by its established owner/name/arity and nominal policy; labels do not select an overload. Match each source argument to one uniquely declared formal slot. Positional arguments fill a leading prefix; reject positionals after the first named argument, missing/unknown/duplicate labels, collisions between positional and named bindings, wrong types, unsupported spreads, absent required slots and ambiguous overloads. A required `T? name` has source type `Option<T>`; it is **not an omittable slot**. The independent `T name?` spelling marks an omittable formal, and `T? name?` combines both properties. Omission is permitted only with a statically checked declaration default (including `None` for an Option slot), never by inventing zero/null or shifting positional slots. Explicit `None` is a supplied typed expression, unlike absent slots or the separate contextual `_`/`omit` markers; any unsupported omission form fails closed. Parameter labels are part of source/override/interface compatibility, not additional runtime dispatch keys.

**Evaluation invariant:** evaluate the callee/receiver, then every supplied source argument expression **exactly once in lexical source order**, validating borrow/move/effects and cancellation along that order. Only after successful evaluation and checking may the compiler permute temporaries into declaration-order ABI slots. A named call cannot duplicate, skip or reorder side effects, or evade type/ownership/permission gates. Compile-time specialization, tail calls, suspendable/resumable methods, constructors and AOT code must consume the same checked plan; unsupported forms fail closed rather than becoming maps/records or dynamically reflected name lookup.

### Named-call edge cases and stable signatures

- **Formal uniqueness and naming:** declarations MUST reject duplicate parameter names, including shadowing introduced by parameter destructuring or receiver aliases. Binding is case-sensitive and against the *resolved callable signature*, not field names or imported aliases. Public callable/interface/override renames that change a named-call label are source-breaking API changes even if their positional ABI stays unchanged; labels cannot select overloads.
- **Arity and overloads:** resolve ordinary candidate sets using established arity/nominal policy before matching labels; do not use a label to rescue or distinguish an otherwise ambiguous overload. Explicit generic instantiation and inferred generic types must agree with the **formal slot selected by the checked value binding**, not source item index.
- **Optional-slot forms:** `T? name` is required `Option<T>`; `T name?` is omittable `T`; `T? name?` is omittable `Option<T>`. A non-Option omitted slot requires an admitted declaration default. Missing, explicitly omitted, and supplied `None` are different checked call items; no expression is evaluated for an omitted slot. A trailing rest formal is not an optional fixed slot.
- **Syntactic ordering:** a positional prefix may precede labeled values; a positional or spread item following the first label is normally rejected. A *single final anonymous trailing closure* is a separately gated exception only if the exact statically proven remaining final formal is callable; it must never be generalized to ordinary positionals or silently accepted by targets without that checked contract.
- **Evaluation and failure:** an invalid binding fails **before** the receiver or any user argument is evaluated. For a valid binding, evaluate the receiver first, followed by supplied argument expressions once in original lexical order. If one expression raises/throws, later expressions are not evaluated; no result permutation may bypass cleanup, permission, borrow/move or effect checking.
- **Indirect/foreign/variadic dispatch:** a closure, external ABI, foreign constructor, protocol dispatch or actor spawn may use labels only when it carries an authenticated static parameter-name map and its dispatcher consumes the identical checked plan. Tuple/record spread expansion and ownership-sensitive forwarding require separate static proof. No runtime `Map`-style rebinding fallback.
- **Diagnostics and tooling:** reject unknown labels, duplicates, collisions with positional bindings, missing required slots, extra arguments, malformed omission and invalid trailing closures deterministically with source spans. The formatter must preserve evaluation order and must not silently turn `name: value` into `name = value`.

**Compatibility boundary:** A readonly record, `Map<K,V>`, tuple spread, structural product, named type metadata and actor message field are not interchangeable named-call argument packs. Static record-to-named forwarding requires an independently checked closed-field expansion with proven evaluation/ownership semantics. Formatters preserve source argument order and labels rather than sorting expressions to declaration order.

## Effect types

A callable can be annotated for potential `throws`, `raises`, `panics` or `suspends` effects. The compile-time effect set is an upper bound used for diagnostics and safe call admission; it is not a license to perform privileged operations. `trap` describes a non-throwing boundary for ordinary throw effects, but does not catch panics or cancellation. Precise declaration syntax and effect-polymorphic inference are still being reconciled.

`async`/`suspend` callables may yield a Future or a Task; the declared carrier type matters. Awaiting a value must not turn a borrowed stack address into a dangling alias or monopolize an OS thread.

## Purity and critical sections

`pure` functions have no externally observable mutation; source effects must obey their declared bound. `critical-read` and `critical-write` are explicit concurrency/lock admission modifiers or callback grants, not promises that the host has acquired a particular OS mutex. A critical write requires exclusive ownership of the protected region; read loans can coexist when no conflicting writer is active.

A critical loan **MUST NOT** leak through a returned pointer, task capture, suspension, actor turn, hot reload, or untrusted domain transition without a separately verified lifetime and capability proof. Await must not hold a blocking host mutex by default.

## Functional pipelines and actor pipelines

```atto
val normalized = raw
  |> trim
  |> parse_value
  |> validate
```

`|>` passes the previous value through a functional transformation.

### Value-match stages

The proposed source contract admits subject-elided value matches as forward
functional stages, with either delimiter style. The result type can be explicit
or inferred, just as in a standalone value match.

```atto
return result
    |> match<int> over
        on Ok(value) -> value,
        on Err(error) -> 42,
    end |> increment;
```

```atto
return result
    |> match over {
        on Ok(value) -> value,
        on Err(error) -> 42,
    } |> increment;
```

For either spelling, `E |> match<T> over ARMS end` (or the brace-delimited
equivalent) is equivalent to `match<T> E over ARMS end`. The preceding expression
is evaluated exactly once, and the selected arm result feeds the next stage.
An explicit second subject is invalid. A standalone match still requires its
own subject. Multiple match stages may compose left-to-right.

The match has the same type inference, exhaustiveness, ordering, pattern-binding
scope, ownership, effects and cleanup semantics as the corresponding standalone
match. The pipeline introduces no implicit copy, borrow, suspension, closure,
or allocation. Both outer delimiter styles admit expression and block arms;
commas between arms and one optional trailing comma are supported. Preserve
existing standalone match separator compatibility. Block-arm returns target the
match result, not the enclosing callable; cleanup completes before the next
stage runs. Nested or early arm-local returns require verified control-flow
and ownership joins: a backend lacking those proofs must reject them rather
than reinterpret them as callable returns.

The formatter preserves the selected outer delimiters and whether the subject
is supplied by the pipeline. This is a source contract proposal, not evidence
of compiler parity. Each backend must independently demonstrate equivalent
typed lowering, positive execution of both Result variants and both delimiters,
continuation after the closing delimiter, one evaluation of effectful subjects,
and negative missing-subject, duplicate-subject, nonexhaustive, ill-typed and
invalid delimiter cases. Unsupported backend subsets must fail closed.

### Unary await stage

```atto
const f2 = 42 |> await;
const f3 = 42 |> await |> identity;
val answer = fetch_data() |> await |> transform;
```

The proposed bare stage `E |> await` is equivalent to ordinary prefix
`await E`. It takes only the immediately preceding expression, evaluates
it once, and exposes its result to the next stage. It is not a callable
factory: `|> await()`, `|> await(other)` and `|> await<T>` are invalid.

For `Future<T>` it awaits one layer and returns `T`. Existing
`HasFuture<T>` projection and actor readiness rules are unchanged. For
a statically proven immediate `int`, both prefix and pipeline forms
return the original integer without creating a Future, suspending, or
changing ownership. The first admission slice is intentionally limited
to `int`; additional immediate types require a static ownership/effect
classification rather than a dynamic catch-all. In particular, do not
silently accept Result, Option, unknown generic, resource or affine
operands as immediate values. Existing `std.await` remains a strict
Future-only adapter. The scalar case may retain conservative suspension
restrictions until the effect/loan analysis can prove it non-suspending.

No implicit lifting of arbitrary stages or recursive flattening is
introduced by the bare operator. Rejection and error propagation must
be the same as prefix await; a backend that cannot provide the effect
semantics rejects before code generation.

### Actor pipelines

The target actor pipeline spelling is:

```atto
val graph = spawn Reader() pipe spawn Transformer() pipe spawn Writer()
```

`pipe` constructs an owned, typed connection graph. Only explicitly compatible actor pipeline roles may join it; supervisor/control messages cannot become data stages; cycle edges require separately bounded feedback channels. This is still a **source-lowering target**, not a claim that the pipeline expression executes on current LLVM/Java main.

The proposed factory spelling is `Actor.pipe(Reader(), Transformer(), Writer())`, with `Actor.pipe(Reader, Transformer, Writer)` as **zero-argument constructor sugar**. Both are compiler-lowered new-actor graph construction, distinct from `Actor.connect(reader_ref, writer_ref)`, which wires already-spawned actor references without adopting them:

```atto
val builder = Actor.pipe(Reader(), Transformer(), Writer())
val with_defaults = Actor.pipe(Reader, Transformer, Writer)
val reader = spawn Reader()
val writer = spawn Writer()
val connection = Actor.connect(reader, writer)
```

**Reconciliation proposal:** `Actor.pipe(...)` returns an **unstarted, single-use builder** and the outer `.start()` attempts transactional actor admission, producing a typed start result (possibly suspendable). The infix `pipe` remains immediate-spawn syntax and shares the same post-start graph semantics. Thus the factory and the infix forms have equivalent constructed topology but **not** identical direct expression return types. Exact `start()` error/effect carriers remain under review. See [factory contract](14-actor-pipeline-factories.md) and [nested pipeline builder hardening](15-nested-actor-pipelines.md). Neither is yet executable backend parity.

## Macros and compile-time evaluation

`define macro` must expand or lower ahead of target runtime execution, be hygienic or explicitly expose controlled capture, produce source-mapped diagnostics, and be deterministic under a pinned toolchain. Macros cannot manufacture permissions or bypass import boundaries. Runtime reflection and runtime code evaluation are not implicit substitutes for macros. Cross-backend macro compilation and behavior require independent executable fixtures.
