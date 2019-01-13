---
layout: post
title:  "const types, traits and implementations in Rust"
date:   2019-01-11 19:02:40 +0000
---
Rust permits a limited form of compile-time function execution in the form of `const` and `const fn`. While, initially, `const` may seem like a reasonaby straightforward feature, it turns out to raise a wealth of interesting and complex design questions. In this post, we're going to look at a particular design question that has been [under discussion](https://github.com/rust-rfcs/const-eval/pull/8) for [some time](https://github.com/rust-rfcs/const-eval/issues/1) and propose a design that is natural and expressive. This is motivated both from a syntactic perspective and a theoretic perspective.

At present, `const fn` is a very restricted form of function. In particular, generic type parameters with trait bounds in any form are not permitted. This is mainly due to the many cases that need consideration when `const` code interacts with run-time code. As we'll see below, this is more complex than one might first think.

This is obviously a desirable feature, but it's hard to be sure that a design meets all the desiderata, while being as minimal as possible. We're going to look at a solution to this problem that should tick all the boxes.

<div class="update" markdown="span">
This post was updated on 2018-01-13 to address the need for [`?const` trait bounds](#opting-out-of-const-trait-bounds-with-const) and default trait implementations.
</div>

This proposed design and post have been coauthored with [@cartesiancat](https://github.com/cartesiancat). Thanks to [@ubsan](https://github.com/ubsan) and [@rpjohnst](https://github.com/rpjohnst) for feedback on an early draft.

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
- To call `foo` at run-time, we must have a run-time value of some type `A` that implements `T`. The implementation of `T for A` may be `const` or not.

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
fn c<A>(A: T) -> bool;
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

If there are default method definitions in the trait, these must either be overridden with `const` method definitions, or the method must be declared `const` in the trait definition (see [below](#explicitly-const-trait-bounds)).

### `const` functions with generic trait bound types
Consider again this previous example:
```rust
const fn baz<A: T>(A) -> A;
```
`foo` may only accept `const` implementations of the trait `T`. Otherwise, it would be possible to write invalid code inside the function body:
```rust
const fn baz<A: T>(A) -> A {
    // `A::foo` might be run-time function here,
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

As mentioned above, any `const` function can also called as a run-time function. Intuitively, by removing the `const` prefix from any function, we get the corresponding run-time definition of the function (as the body is entirely unmodified).

### Explicitly-`const` trait bounds
Take the following run-time function signature:
```rust
fn baz<A: const T>(A) -> A;
```
This syntax means `A` is explicitly required to `const` implement `T`. When is it useful for users to be able to explicitly declare trait bounds `const`?

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

In the proposed design, it necessary to explicitly declare `const` bounds on `const` functions when those traits are made use of inside `const` definitions, so that they remain valid when converted to run-time functions. For example, `const fn baz<A: const T>(A) -> A` will always take only `const` impls for `A`, whether called at compile-time or run-time.

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

### Opting out of `const` trait bounds with `?const`
There's one more ability we would like to be completely flexible with the strictness of our trait bounds (and to avoid requiring any duplication of trait definitions in some situations).

Trait bounds in `const` functions require `const` implementations by default, which matches the intuition for run-time functions: "if you have a parameter with a trait bound `T`, you know that all the requirements of the bound can be used inside the function". However, sometimes you don't need such a strong restriction. Recall how, with `const` declarations in trait definitions, we could avoid having `const` trait bounds in run-time functions, as long as every method we used in a `const` context was declared `const` in the trait. Equally, we would like this ability in `const fn`.

For example, take the following:
```rust
trait T {
    const fn choice() -> bool;

    fn validate(u8) -> bool;
}

struct S;

impl T for S {
    const fn choice() -> bool {
        ...
    }

    fn validate(u8) -> bool {
        ...
    }
}

const fn bar<A: T>(A) -> A {
    let x: bool = <A as T>::choice();
    ...
}

// We can't call `bar` with a value of `S`, because
// `S` doesn't `const`-implement `T`, even though it
// only makes use of `const` functions!
```

We would like some way to relax this requirement when necessary. This is achieved by means of the explicit `const` trait bound opt-out: `?const`.

```rust
// ...continuing the previous example...

const fn bar_opt_ct<A: ?const T>(A) -> A {
    let x: bool = <A as T>::choice();
    ...
}

// We can call `bar_opt_ct` with a value of `S`, because
// the only method it makes use of is declared `const`
// in the trait `T`.
```

The `?const` syntax mirrors that for `?Sized`, as an opt-out of the default (most common) behaviour. With this keyword, one now has full expressivity over trait bounds.

- By default, `const fn` will require `const` trait bounds, so that you can freely use the trait within the function. At run-time, such functions have no restrictions on the trait bounds.
- Trait bounds prefixed by `const` act like normal at compile-time, but also require `const` trait bounds at run-time.
- Trait bounds prefixed by `?const` do not require `const` trait bounds, at compile-time or at run-time.
- Methods may be called in a `const` context (such as at compile-time, or in an inner `const` at run-time) if either they originate from a `const` trait bound, or if they are explicitly declared `const` in the trait.

### Removal of the `const` keyword
Since any `const` function can be called at run-time, it must also be a valid non-`const` function (after a suitable translation): this is what gives the intuition and motivation for our definition. The translation simply modifies the function signature without changing the body. This translation is extremely simple and involves simply removing the `const` prefix from a function and removing any `?const` bounds.

```rust
trait T {
    const fn choice() -> bool;
    ...
}

// This function at compile-time...
const fn baz_ct<A: ?const T>(A) -> A {
    let x: bool = <A as T>::choice();
    ...
}

// ...is equivalent to this function at run-time.
fn baz_rt<A: T>(A) -> A {
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
// function.
const fn bop_ct<A: T>(A) -> A {
    const X: bool = <A as T>::choice();
    ...
}

// `bot_ct` is equivalent to this function.
fn bop_rt<A: T>(A) -> A {
    // This is not OK, because we have no assurance
    // that `choice` is a `const fn`.
    const X: bool = <A as T>::choice(); // ERROR!
    ...
}
```

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

#### `const impl` versus `impl const`
We have a choice of syntaxes for the `const` implementation syntax sugar, both of which are consistent (in different ways) with other similar syntaxes.

`const impl` is consistent with the practice of prefixing `impl` with modifiers (e.g. `default`, `unsafe`) and prefixing const items with `const` (e.g. `const fn` and, in this proposal, `const trait`).

`impl const` is consistent with the syntax in this proposal used for `const` trait bounds, where trait names are prefixed with `const`.

The former choice seems slightly more justified by existing syntax, but either is a viable option from a consistency perspective.

### Inherent implementations
Note that while inherent implementations receive the same `const`-prefixing syntax as trait implementations, the notion of "`const` inherent implementation" does not apply. Inherent functions may be called in `const` code when they themselves are `const`. `const` inherent functions are converted to run-time functions in the same way as any other `const` function.

### `const` and subtyping
In the above discussion, we've talked about `const fn` from the perspective of being "equivalent to" or "converted to" a run-time `fn`, when called at run-time. This is one way to consider `const` functions' relation to run-time functions, but not the only one.

Alternatively, we may view `const` function types as being subtypes of run-time function types. In this light, a `const fn` type is a subtype of the run-time function type that we've described it as being "converted" to. Values of subtypes are simply particular cases of their parent types, which makes it evident that `const` functions should be callable at run-time.

For example[^signature-type-notation]:
- `const fn foo(A) -> B` is a subtype of `fn foo(A) -> B`.
- `const fn bar<A: T>(A) -> B` is a subtype of `fn bar<A: T>(A) -> B`.
- `const fn bop<A: const T>(A) -> B` is a subtype of `fn bar<A: const T>(A) -> B`.

[^signature-type-notation]: Technically, I'm abusing notation here, as function signatures aren't actually types. The type of `const fn foo(A) -> B` should properly be written `const fn(A) -> B {foo}`. In the name of readability, I'm going to pretend signatures are types (signatures uniquely determine types, so this is unambiguous).

We'll briefly touch on why these are equivalent ways to view "`const` at run-time" in the theoretic model below.

## Summary
As far as we're aware, this encompasses all use cases for generic `const` functions with trait bounds in a syntactically minimal and natural manner and hopefully this is reflected in the design. For the most part, users should not have to worry about where to place `const`, but the rules of behaviour are straightforward even in more complex scenarios.

This ends the design of the feature, but before finishing, we're going to briefly see that this design reflects a sound theoretic model of `const` types, which provides more confidence in the correctness of the design.

## A category theoretic model for `const`
From a practical standpoint, the syntax proposed we've proposed seems most natural and expressive. But it can be easy to overlook aspects of a new feature in a programming language, especially when it interacts with the type system. To be fully justified in a new design, it is extremely valuable to have a (type or category) theoretic model, which is a precise mathematical description of the types and their interactions. Indeed, it was from the theoretic model that led us to ultimately settle on this choice of syntax for the proposal.

Here, we're going to briefly outline what this design corresponds to in a naïve category theoretic model of Rust. This should give a basic intuition for why this is a natural design from a theoretic viewpoint as well as a practical one. A full treatment of Rust's type system in general, and with respect to `const`, is left for a future occasion.

This is intended as a sketch for those with some familiarity with category theory: understanding it isn't critical to the understanding of the proposed design. ([Equivalently](https://en.wikipedia.org/wiki/Curry%E2%80%93Howard_correspondence#Curry%E2%80%93Howard%E2%80%93Lambek_correspondence), one could consider this design from the perspective of type theory.)

### The universe of types
The collection of all types in Rust, together with the collection of all functions[^fn-notation] and the obvious notions of composition (namely, function composition) and identities (any notion of identity function, such as `|x| x`), forms a [category](https://en.wikipedia.org/wiki/Category_(mathematics)).

[^fn-notation]: Here, the function type `A -> B` is taken to be any type that implements a corresponding `Fn*` trait, for example `Fn(A) -> B`, `FnMut(A) -> B` and `FnOnce(A) -> B`.

This universe contains the usual types, such as `()`, `bool`, `u8` and user-defined types. However, it also contains *another* version of each of the types, corresponding to `const`. For example, when we write:
```rust
const X: bool = false;
```
`X` implicitly has type `const bool`. All (non-function) user-defined and primitive types have `const` analogues.

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

Of course, we're able to use `const` types within run-time functions. Implicitly, this makes use of a coercion from `const` types to non-`const` types at run-time. The coercion cannot be applied at compile-time (or, equivalently, in `const` contexts), which is why the `const A` in the example above cannot simply be coerced to a run-time `A` to call `foo`. The coercion of most types is trivial, but the coercion of `const` function types in particular is given by the rules described in the previous section.

### The *unconst* monad
There is a canonical operation that transforms a `const` type into a run-time type. Inuitively, this corresponds to removing the `const` prefix from any type (and potentially adding explicit `const` modifiers to each trait bound). Let's call this operation `U` (for *unconst*). `U` acts on both types (the objects of the universe `Type`) and functions (conserving their definitions while converting `const fn` to `fn`). Run-time types are fix-points for`U`: applying it on a non-`const` type has no effect. `U` is a functorial operation (following directly from the definitions) and hence forms an endofunctor on the category `Type`.

What's more, given any `const` type, we have a trivial function taking any value thereof to that of the run-time value[^const-generics]. Applying `U` twice is the same as applying it once (due to the fix-point observation above), so we have a trivial isomorphism for any type `A`, from `U(U(A))` to `U(A)`. Together, these functions give `U` the structure of an [idempotent monad](https://ncatlab.org/nlab/show/idempotent+monad) on `Type`[^monad-presentation][^monad-motivation].

[^const-generics]: If our model of the types in Rust includes const generic functions, this function can be explicitly described as a Rust function; otherwise it simply lives in our metatheory.

[^monad-presentation]: We've presented a monad here from the perspective of [a monad as a monoid](https://en.wikipedia.org/wiki/Monad_(category_theory)#Formal_definition). Reformulating it [in terms of Kleisli maps](https://ncatlab.org/nlab/show/extension+system) may be more familiar to a functionally-oriented programmer and is left as an exercise to the reader.

[^monad-motivation]: For those of you wondering, we don't *need* `U` to be a monad to construct this model: it would work similarly well if `U` was simply a functor (or even a type-level function). But monads are a lot more fun.

In Rust, practically speaking, this *unconst* monad `U` is applied as an implicit coercion whenever a value of a `const` type is used as a non-`const` type, or a `const fn` is called at run-time.

#### Subtyping
When viewed as an implicit coercion, `U` reflects the perspective of "`const fn` is converted at run-time to `fn`". We can also view it from the perspective of "`const fn`s are subtypes of `fn`s".

Here, we may define a [partial order](https://en.wikipedia.org/wiki/Partially_ordered_set) on types such that a type `A < B`  if `A = B` or `U(A) = B`. It is easy to see this has the required properties. This partial order provides a subtyping relation on types: `A: B` if `A < B`. The transitive closure of this partial order relation with the existing subtyping relation between types of the same `const`ness gives us the full model of subtyping arising from the proposed design.

## Wrapping up
This theoretic model is simple, but provides some justification for the definitions of the coercions described above. The rich structure on the coercion indicates that, at least from a theoretic perspective, this is quite a natural choice. Some of the previous drafts of `const` implementations, bounds, etc. have not been so naturally expressible theoretically; so while a simple one, the existence of such a model can be an effective litmus test.

This design will certainly require more discussion, but we hope that this, or a similar proposal, will make its appearance as a [new RFC](https://github.com/rust-lang/rfcs/pulls) in the not too distant future.
