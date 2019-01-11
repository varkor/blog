---
layout: post
title:  "const types, traits and implementations in Rust"
date:   2019-01-11 00:31:08 +0000
---
Rust permits a limited form of compile-time function execution in the form of `const` and `const fn`. While, initially, `const` may seem like a reasonaby straightforward feature, it turns out to raise a wealth of interesting and complex design questions. In this post, we're going to look at a particular design question that has been under discussion for some time and propose a design that is natural and expressive. This is motivated both from a syntactic perspective and a theoretic perspective.

At present, `const fn` is a very restricted form of function. In particular, generic type parameters with trait bounds in any form are not permitted. This is mainly due to the many cases that need consideration when `const` code interacts with run-time code. As we'll see below, this is more complex than one might first think.

This is obviously a desirable feature, but it's hard to be sure that a design meets all the desiderata, while being as minimal as possible. We're going to look at a solution to this problem that should tick all the boxes.

This proposed design and post have been coauthored with a fellow type enthusiast.

## Proposed design
The most important concept to get right when dealing with `const` types, traits and implementations is the question of how const functions are treated as, or converted to, runtime functions. We should always able to call const functions at runtime, with the most permissive set of rules on their arguments. The rules determining this behaviour should feel natural (users shouldn't usually have to explicitly think about them), but the explicit rules should also be straightforward.

The syntax proposed here is, we think, the simplest and most permissive syntax, while being consistent with the existing syntax.

First, let's take a look at the syntax and see some examples of the conversions from `const fn` to `fn`.

Take the following `const fn` declaration:
```rust
const fn foo<A: T>(A) -> A;
```
This is interpreted in the following manner:
- To call `foo` at compile-time, we must have a `const` value of some type `A` that implements `T`. Importantly, the implementation of `T for A` must itself be `const`. (We shall see exactly what a "`const` implementation" is soon.)
- To call `foo` at run-time, we must have a run-time value of some type `A` that implements `T`. For consistency, the implementation of `T for A` must still be `const`. (Though this choice is discussed in more detail below.)

Analogously, take the following run-time `fn` declaration:
```rust
fn bar<A: T>(A) -> A;
```
This is interpreted in the following manner:
- `bar` cannot be called at compile-time.
- To call `bar` at run-time, we must have a run-time value of some type `A` that implements `T`. The implementation of `T for A` may be `const` or not.

Here are some examples of `const` functions and their run-time analogues.

```rust
const fn a(u8) -> bool;
// ...when called at runtime is equivalent to...
fn a(u8) -> bool;

const fn b<A>(A) -> bool;
// ...when called at runtime is equivalent to...
fn b<A>(A) -> bool;

const fn c<A: T>(A) -> bool;
// ...when called at runtime is equivalent to...
fn c<A>(A: const T) -> bool;
```

### `const` implementations
A `const` implementation of a trait `T` for a type `A` is an implementation of `T` for `A` such that every function is a `const fn`.

```rust
struct C; struct D; struct E; struct F;

trait T {
    fn foo(C) -> D;
    fn bar(E) -> F;
}

struct Q;

// This is a "non-`const`" implementation of `T` for `Q`.
impl T for Q {
    fn foo(c: C) -> D { ... }
    fn bar(e: E) -> F { ... }
}

struct R;

// This is a "non-`const`" implementation of `T` for `R`.
impl T for R {
    fn foo(c: C) -> D { ... }
    const fn bar(e: E) -> F { ... }
}

struct S;

// This is a `const` implementation of T for S.
// An implementation is a `const` implementation
// iff all functions within are `const`.
impl T for S {
    const fn foo(c: C) -> D { ... }
    const fn bar(e: E) -> F { ... }
}
```

Any implementation containing any non-`const` functions is not a `const` implementation, e.g. those for `Q` and `R` in the examples above.

### `const` functions with generic trait bound types
Consider again this previous example:
```rust
const fn baz<A: T>(A) -> A;
```
`foo` may only accept `const` implementations of the trait `T`. Otherwise, it would be possible to write invalid code inside the function body:
```rust
const fn baz<A: T>(A) -> A {
    // `A:foo` might be run-time function here,
    // which we cannot call at compile-time!
    let x: D = A::foo(...); // ERROR!
    ...
}
```
Therefore, in general, any `const` function definition of the form
```rust
const fn bop<A1: T1, ..., An: Tn>(...) {
    ...
}
```
may take only `const` implementations for each of the traits `T1, ..., Tn`.

As mentioned above, any `const` function can also called as a run-time function, having the same definition and effectively having *almost* the same signature (minus the leading `const`).

Why "almost"? Well, we have a choice here as to the semantics of `const` functions at run-time.

#### Option A. `const fn` at run-time requires `const` implementations
Under this choice, we have the following analogue of `const fn` to `fn` at run-time:
```rust
const fn baz<A: T>(A) -> A;
// ...when called at runtime is equivalent to...
fn baz<A: const T>(A) -> A;
```
This "`const` trait bound" syntax means that `A` must always `const`-implement `T`, both for when it is called at compile-time and run-time.

Therefore, calling a `const` function, even at run-time, never transitively calls a non-`const` function. This is perhaps the most intuitive behaviour. For instance, while `const` functions are pure, it means that calling a `const` function such as `baz` above will always be a pure operation, even at run-time.

#### Option B. `const fn` at run-time permits any implementation
Under this choice, we have the following analogue of `const fn` to `fn` at run-time:
```rust
const fn baz<A: T>(A) -> A;
// ...when called at runtime is equivalent to...
fn baz<A: T>(A) -> A;
```
Here, when we call `baz` at run-time, we may pass any type `A` that implements `T` in any fashion, even if it not a `const` implementation. This is more expressive, but leads to the slightly unintuitive property that `const` functions may call non-`const` functions at run-time via trait methods.

We are proposing **Option A** in this design. Ultimately, it is forwards-compatible with extending the design to **Option B** in the future, so this is the most pragmatic and conservative choice.

Intuitively, note that removing the `const` prefix from any function (and moving the `const` to any trait bounds) gives the corresponding run-time definition of the function.

### Explicitly-`const` trait bounds
Take the following run-time function signature:
```rust
fn baz<A: const T>(A) -> A;
```
This syntax means `A` is explicitly required to `const` implement `T`. We've already seen `const` being used explicitly in trait bounds, but as mentioned there, it's not strictly necessary in those examples. When is it useful for users to be able to explicitly declare trait bounds `const`?

Specifically, explicit `const` trait bounds are necessary when run-time functions contain `const` code. Here's a simple example:
```rust
fn baz<A: const T>(A) -> A {
    // We can only call a `T` method of `A`
    // in a `const` variable declaration
    // if we know `A` `const`-implements `T`,
    // so the trait bound must explicitly
    // be `const`.
    const X: bool = <A as T>::choice();
    ...
}
```

In the proposed design, it is never necessary to explicitly declare `const` bounds on `const` functions[^option-b-note].

[^option-b-note]: However, under the **Option B** design, explicit `const` trait bounds would be required on `const fn` trait bounds if they made use of those traits inside `const` definitions, to remain valid when converted to a run-time function. That is to say that, even under __Option B__, `const fn baz<A: const T>(A) -> A` will always take only `const` impls for `A`, whether called at compile-time or run-time.

### `const` in traits
In a trait declaration we can place `const` in front of any function declaration to require that all implementations must define that function as `const`. Consider again the previous example:
```rust
trait T {
    const fn choice() -> bool;
    ...
}
fn baz<A: T>(A) -> A {
    // Now, `<A: const T>` is not needed, since
    // `choice` is always const in any implementation
    // of `T`.
    const X: bool = <A as T>::choice();
    ...
}
```

### Removal of the `const` keyword
Since any `const` function can be called at run-time, it must also be a valid non-`const` function (after a suitable translation): this is what gives the intuition and motivation for our definition. The translation simply modifies the function signature without changing the body. These translations are slightly different under the choice of **Option A** or **Option B**.

```rust
trait T {
    fn choice() -> bool;
    ...
}

const fn baz<A: T>(A) -> A {
    let x: bool = <A as T>::choice();
    ...
}

// Option A: move `const` to all trait bounds.
fn bazA<A: const T>(A) -> A {
    let x: bool = <A as T>::choice();
    ...
}

// Option B: simply remove `const`.
fn bazB<A: T>(A) -> A {
    let x: bool = <A as T>::choice();
    ...
}
```

Recall that if a method in the trait is not declared `const` then it cannot be used inside a `const` definition in the body of a function (regardless of whether that function is `const` or non-`const`).
```rust
trait T {
    fn choice() -> bool;
    ...
}

// The following function is not permitted, as its
// run-time translation is not a valid run-time
// function under Option B.
const fn bop<A: T>(A) -> A {
    const X: bool = <A as T>::choice();
    ...
}

// Option A
fn bopA<A: const T>(A) -> A {
    // This is OK, as the trait bound is explicitly `const`.
    const X: bool = <A as T>::choice();
    ...
}

// Option B
fn bopB<A: T>(A) -> A {
    // This is not OK, because we have no assurance
    // that `choice` is a `const fn`.
    const X: bool = <A as T>::choice(); // ERROR!
    ...
}
```

Technically, although omitting an explicit `const` trait bound is compatible with **Option A**, it is not forwards-compatible with **Option B** and so is disallowed for now.

### Syntactic sugar for `const` on `trait`s and `impl`s
For the common practice of declaring `const` every method in an `impl`, or in a trait, we have the following syntactic sugar. Prefixing `impl` or `trait` with `const` amounts to prefixing every function declaration and definition with `const`.

```rust
const trait V {
    fn foo(C) -> D;
    fn bar(E) -> F;
}
// ...desugars to...
trait V {
    const fn foo(C) -> D;
    const fn bar(E) -> F;
}

struct P;

const impl V for P {
    fn foo(C) -> D;
    fn bar(E) -> F;
}
// ...desugars to...
impl V for P {
    const fn foo(C) -> D;
    const fn bar(E) -> F;
}
```

Note that the syntactic sugar for traits, `const trait`, is consistent with the explicit `const` trait bounds on generic type parameters. In both cases, a `const`-prefix implies that all trait methods must be `const`.

#### API considerations
When `const`-prefixing is simply syntactic sugar, it may be easy to accidentally change the `const`-ness of an implementation by changing a single function. It thus may be desirable to only consider an implementation `const` if it is prefixed with `const`. That way, the only way implementations may be converted between `const` and non-`const` is by explicitly adding or removing the `const` prefix. For now, whether this a sensible design choice is left as an open question.

### Inherent implementations
Note that while inherent implementations receive the same `const`-prefixing syntax as trait implementations, the notion of "`const` inherent implementation" does not apply. Inherent functions may be called in `const` code when they themselves are `const`. `const` inherent functions are converted to run-time functions in the same way as any other `const` function.

## Summary
As far as we're aware, this encompasses all use cases for generic `const` functions with trait bounds in a syntactically minimal and natural manner and hopefully this is reflected in the design. For the most part, users should not have to worry about where to place `const`, but the rules of behaviour are straightforward even in more complex scenarios.

This ends the design of the feature, but before finishing, we're going to briefly see that this design reflects a sound theoretic model of `const` types, which provides more confidence in the correctness of the design.

## A type theoretic model for `const`
From a practical standpoint, the syntax proposed we've proposed seems most natural and expressive. But it can be easy to overlook aspects of a new feature in a programming language, especially when it interacts with the type system. To be fully justified in a new design, it is extremely valuable to have a type theoretic model, which is a precise mathematical description of the types and their interactions. Indeed, it was from the theoretical model that led us to ultimately settle on this choice of syntax for the proposal.

Here, we're going to briefly outline what this design corresponds to in a naïve type theoretic model of Rust. This should give a basic intuition for why this is a natural design from a theoretic viewpoint as well as a practical one. A full treatment of Rust's type system in general, and with respect to `const`, is left for a future occasion.

This is intended as a sketch for those with some familiarity with type theory: understanding it isn't critical to the understanding of the proposed design.

### The universe of types
The collection of all types in Rust, together with the collection of all functions[^1] and the obvious notions of composition (namely, function composition) and identities (any notion of identity function, such as `|x| x`), forms a [category](https://en.wikipedia.org/wiki/Category_(mathematics)).

[^1]: Here, the function type `A -> B` is taken to be any type that implements a corresponding `Fn*` trait, for example `Fn(A) -> B`, `FnMut(A) -> B` and `FnOnce(A) -> B`.

This universe contains the usual types, such as `()`, `bool`, `u8` and user-defined types. However, it also contains *another* version of each of the types, corresponding to `const`. For example, when we write:
```rust
const X: bool = false;
```
`X` implicitly has type `const u8`. All (non-function) user-defined and primitive types have `const` analogues.

Within the scope of a `const fn`, all values have `const` types.
```rust
struct A;
struct B;

const fn foo(a: A) -> (A, B) {
    // a: const A
    let b = B; // b: const B
    (a, b)     // (a, b): const (A, B)
}
```

Note that this gives us the reason why run-time functions may not be called at compile-time: they simply cannot provide the correct input types.
```rust
struct A;

fn foo(a: A) { ... }

const fn bar(a: A) {
    foo(a) // ERROR! `foo` expects a (run-time) `A`,
           // but we've given it a `const A`.
}
```

Of course, we're able to use `const` types within run-time functions. Implicitly, this makes use of a coercion from `const` types to non-`const` types. The coercion of most types is trivial, but the coercion of `const` function types in particular is given by the rules described in the previous section.

### The *unconst* monad
There is a canonical operation that transforms a `const` type into a run-time type. Inuitively, this corresponds to removing the `const` prefix from any type (and potentially adding explicit `const` modifiers to each trait bound). Let's call this operation `U` (for *unconst*). `U` acts on both types (the objects of the universe `Type`) and functions (conserving their definitions while converting `const fn` to `fn`). Run-time types are fix-points for`U`: applying it on a non-`const` type has no effect. `U` is a functorial operation (following directly from the definitions) and hence forms an endofunctor on the category `Type`.

What's more, given any `const` type, we have a trivial function taking any value thereof to that of the run-time value[^const-generics]. Applying `U` twice is the same as applying it once (due to the fix-point observation above), so we have a trivial isomorphism for any type `A`, from `U(U(A))` to `U(A)`. Together, these functions give `U` the structure of an [idempotent monad](https://ncatlab.org/nlab/show/idempotent+monad) on `Type`.

[^const-generics]: If our model of the types in Rust includes const generic functions, this function can be explicitly described as a Rust function; otherwise it simply lives in our metatheory.

In Rust, practically speaking, this *unconst* monad `U` is applied as an implicit coercion whenever a value of a `const` type is used as a non-`const` type, or a `const fn` is called at run-time.

## Wrapping up
This theoretic model is simple, but provides some justification for the definitions of the coercions described above. The rich structure on the coercion indicates that, at least from a theoretic perspective, this is quite a natural choice. Some of the previous drafts of `const` implementations, bounds, etc. have not been so naturally expressible theoretically; so while a simple one, the existence of such a model can be an effective litmus test.

This design will certainly require more discussion, but we hope that this, or a similar proposal, will make its appearance as a [new RFC](https://github.com/rust-lang/rfcs/pulls) in the not too distant future.
