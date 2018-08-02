---
layout: post
title:  "Existential types in Rust"
date:   2018-07-03 22:25:50 +0100
---
Existential types are a hot topic in Rust at the moment. In Rust 1.26, a feature was stabilised known as `impl Trait`. This addition was declared a long-awaited syntax for existential types, but its inclusion was not without some controversy.
There seems to be a lot of confusion as to just what `impl Trait` really means from a type theoretic (viz. *precise*) perspective, even two months after its stabilisation, and I don't think anyone has really captured exactly what `impl Trait` is and isn't. This is the result of several facets of the design, each of which compounds the bewilderment.

But there's another feature that rarely gets mentioned in the same setting, which also contributes some form of existentially-quantified type to Rust, known as `dyn Trait`, and it's not possible to really understand the situation without taking it into account.

In addition to all of this, there have been several RFCs proposing extensions to the state of affairs. I think that, without properly understanding exactly what each syntax means at the moment, it's going to be far too easy to make mistakes with new syntax — so we need to make sure we're very careful about what we're proposing.

This post is intended to clear up hopefully most of the confusion around `impl Trait` and existential types in general in Rust, and provide some guidance in particular for one ongoing discussion around existential type aliases.

I'll be trying to explain the concepts here without recourse to too much type theory to hopefully make it accessible.

## Preface: what is an existential type
An "existential type", or "existentially-quantified type", is a type that intuitively represents "any type satisfying a given property". In the context of Rust, this usually means "any type implementing a given trait". I'll use a notation from logic to describe an existential type here (which I think is slightly clearer than the type theoretic notation).

`∃ T. T: Trait` is the logical notation for *some* type `T` such that `T` implements the trait `Trait`.
For example, `∃ T. T: IntoIterator<Item = S>` could be satisfied with the type `Vec<S>` or `HashSet<S>`, or so on.

We'll also talk about universally-quantified types in passing too, so let's clarify the notation for that too. The type `∀ T. (T, T)` is a universally-quantified type that takes a type argument `T` and produces a type of pairs of that type, `(T, T)`. You can think of `∀ T. S` as a function taking a type `T` and returning a type `S`.

## What does `impl Trait` *mean*?
Naïvely, any time you see the type `impl Trait`, you can substitute it with an existential type `∃ T. T: Trait` — that is, there exists some type `T` such that `T` implements the trait `Trait`. This existential quantification binds tightly, so `fn foo(a: A, b: B, c: C) -> impl Bar` is the syntax for a type `fn(A, B, C) -> (∃ T. T: Bar)`.

However, this _isn't_ quite true, though it's not for the reason many people think. To explain it, let's quickly talk about why people find `impl Trait` so troublesome.

## Confusion 1: Universally-quantified type syntax vs existentially-quantified type syntax
Rust has had universally-quantified types for a long time, in the form of generic parameters. However, there's a couple of differences between the syntax for existentially-quantified types and universally-quantified types that are easy to overlook at first.

`fn foo<S, T>(s: S, t: T) -> T` is syntax for a type `∀ S, T. (fn(S, T) -> T)` — that is, a function taking two type arguments (`S` and `T`) and two values of those respective types and returning a value of type `T`. Notice the difference here with the existential type: the universal quantifier `∀` is scoped over *the entire type* — everything after the `∀` — whereas the existential quantifier `∃` is scoped over *just the immediate trait bound* (looking at where the quantifier is placed, in `fn(A, B, C) -> (∃ T. T: Bar)` the `∃` is after the function arrow `->`, whereas in  `∀ S, T. (fn(S, T) -> T)` the `∀` is at the outermost level). `∃` is, in some sense, much more local. The syntax is indicative of this, with type parameters being placed at the beginning of the definition, in contrast to `impl Trait`, which is placed inline.

It's useful to bear in mind that one key difference between universal quantification (generic parameters) and existential quantification is this scoping difference. (Be reassured that this difference is entirely intentional. If you try to think what it would mean if their scopes were different, you'll probably realise that it makes things a lot harder to make sense of.)

## Confusion 2: Return-position `impl Trait` vs argument-position `impl Trait`
The supposed distinction, or lack thereof, between `impl Trait` in different positions, is a very contentious issue. This is unsurprising, as even official sources have made conflicting assertions about what `impl Trait` really means in argument position.

One quick side note before we begin: I'm using RPIT (return-position `impl Trait`) and APIT (argument-position `impl Trait`) as initialisms, because typing out the whole phrases each time gets old quickly. Okay. Let's take a look at the issue. Say we have two functions signatures:
```rust
fn foo() -> impl Foo; // Return-position `impl Trait` (RPIT)
fn bar(impl Bar) -> (); // Argument-position `impl Trait` (APIT)
```
The question is thus: if the return-position `impl Foo` is an existential type, as proposed earlier, what does that make the argument-position `impl Bar`?

To answer that, let's apply our syntax-to-type transformation: `fn bar(impl Bar) -> ()` is a type `(∃ T. T: Bar) -> ()`. This is a function that takes a value of *some type*, given that the type implements the trait `Bar`. The type `T` itself is not a parameter — only the value is.

We're going to have to take a slight diversion into type theory here, because it motivates a result that is perhaps intuitive. The following proposition holds in intuitionistic logic: `((∃ x. P(x)) → Q) ⇔ (∀ x. (P(x) → Q))`, which means that, according to the Curry–Howard Correspondence, it also holds when considering the proposition as a type.

What does this mean for us? It means that if we have a function type `fn(∃ S. S: Foo) → T`, then we can construct a new type `∀ S. (fn(S: Foo) -> T)` (and vice versa). But hey, look at that! If we convert the type to our Rust syntax, the first is a function `fn(impl Foo) -> T` and the second is a function `fn<S: Foo>(S) -> T`. So while the two function types are *not* the same (in terms of being identical), they are isomorphic — and we can freely convert between them.

(We can tell they're not exactly the same, because we can distinguish between `fn(impl Foo)` and `fn<S: Foo>(S)` by trying to pass a type argument to both. We _can_ pass a type argument to the latter, universally-quantified function, because `S` is actually a parameter (it's just usually inferred by Rust) — using the "turbofish" notation `::<S>`. But we cannot pass a type argument to the former, because it doesn't have a type parameter — it simply takes a value whose type implements the trait `Foo`.)

In summary: `impl Trait` is *always existential*. It's not universal in argument position — it's just conveniently isomorphic to a very similar universally-quantified type.

## There's a "but"...
There's still something fishy about `impl Trait` though, which is why the story isn't quite finished here. Consider the following:
```rust
trait Trait {}
struct A;
struct B;
impl Trait for A {}
impl Trait for B {}

fn foo(pick_a: bool) -> impl Trait {
	if pick_a { A } else { B } // ERROR: if and else have incompatible types
}
```
If `impl Trait` just represented an existential type, this should work. `A` is certainly a value of a type that implements `Trait`. So is `B`. So what's the problem?

The problem is that the Rust compiler needs to know which (unquantified) type will be returned from the function. The existential type doesn't "exist" at run-time — it needs to pick a specific unquantified type. (This makes returning an `impl Trait` just as efficient as any other type.)

But this is a pretty big restriction. In fact, it changes the semantics of our existentials. For each occurrence of `impl Trait`, we only allow a single unquantified type to represent it. Specifically, for return-position `impl Trait`, the existential quantifier is over the entire function (though within the universal quantifiers) — it can only ever have one instantiation. However, argument-position `impl Trait` really is existential at the correct, tightly-binding scope.

...Which means we now have a distinction between the two. Though argument-position and return-position `impl Trait` are both *forms* of existential quantification, they're over different scopes: APIT is tightly-bound, whereas RPIT is bound at the level of the whole function.

## Hope for existentials
I hinted at something in the introduction, though, which if left unmentioned would leave our story incomplete. There's *another* type that provides some form of existential quantification: `dyn Trait`. In present-day Rust, `dyn Trait` has to be boxed — that is, placed behind a pointer of some kind, like `&` or `Box<>`. But if we hypothetically extend the concept to an unboxed `dyn Trait`, we notice that it has precisely the semantics we'd expect of an existential type. In fact, the type for `dyn Trait` is the same as that of argument-position `impl Trait`: `(∃ T. T: Trait)`. The difference is in the representation (in the compiler): because `dyn Trait` is a true existential, it has to hold information about the specific type `T` *at run-time*, which is a disadvantage that `impl Trait` doesn't have. In theory, though, `impl Trait` is just a more restricted (albeit unboxed) version of `dyn Trait`, which you can intuitively think of as having some guaranteed compiler optimisations (like not having to store type information).

## Take-aways
Even if you didn't follow all of the details above, you can take away a few key points, which should clear up some of the confusion surrounding `impl Trait`:
- Argument-position `impl Trait` is *not* universally-quantified, but it acts very similarly (due to neat type theory equivalences!).
- However, APIT and RPIT *are* semantically different. The difference rather comes down to quantification scope.

## Syntax for existential type declarations
If you aren't already very familiar with how much confusion existential types has created, the above has probably given you some idea. I think this demonstrates that we need to be *really* careful with any additional syntax we propose for existential types (or similar), as the concepts are quite subtle, but can have huge consequences for the design of the language. To show this isn't just an idle discussion, I'm going to take a look at a proposed feature for defining existential types, outside of inline use in function signatures.

There are currently two proposed and widely-supported syntaxes for declaring an existential type. However, I propose that these are both inconsistent with Rust's syntax and semantics, and we need to take a different tack to solve the presented problem.

Before we look at the proposals, note what problem we're trying to fix. We want to declare a type that is the *same* wherever it is placed, but doesn't require explicit mention of what the type is, as long as it satisfies some bound. We want something like RPIT as a standalone type (which is not currently possible).

### Proposal 1: `type Foo = impl Bar`
This proposal is put forward as the "intuitive" syntax for defining an existential type. It's not as rosy as it first seems, however.

The syntax `type A = B` is type alias syntax. It gives a new name for an existing type. So if we expected `type Foo = impl Bar` to be valid, we'd expect it to be a type alias. Let's see how this could work.

`type Foo = impl Bar` indicates that `impl Bar` is a type. But what type is it? It could be a new (hidden) type `∃ T. T Bar` (like argument-position `impl Trait`). But then you wouldn't be able to use it as the return type of a function. Or it could be a new existential type quantified over the entire program (like return-position `impl Trait`). But then you wouldn't be able to use it as an argument type of a function.

The point is that `impl Trait` *is* contextual. It doesn't represent a single type in isolation. So `type Foo = impl Bar` is ambiguous syntax; it's no longer an alias.

You could suggest overloading `type Foo = impl Bar` to be a particular existential type declaration (such as return-position `impl Trait`), but now `type Foo = /* ... */;` isn't consistent: it might be a type alias or a type definition (this breaks what people refer to as "referential transparency" — you can't just use `Foo` anywhere you would use `impl Bar`).

Alternatively, you could suggest that a type alias of a contextual type is also contextual. This completely changes the semantics of type aliases and I can't imagine anyone suggesting this would be a good idea, as it breaks a lot of assumptions people have about aliases (as opposed to `impl Trait`, which is arguably syntactic sugar for something that doesn't actually exist in Rust yet).

### Proposal 2: `type Foo: Bar`
This proposal is put forward as an alternative to the above syntax, as syntax in which we declare a type, but don't specify it. This is also problematic.

The syntax `type A: B` is associated type syntax. (In fact, associated types are often brought up with respect to this syntax proposal.) But associated types are *not* existential types: they're (a restricted form* of) type parameters. By introducing a syntax `type Foo: Bar` for existential types, you now no longer have consistency with regards to what the syntax means. Just like `impl Trait`, it's suddenly, confusingly, contextual. Inconsistencies in Rust's syntax (though thankfully these are relatively few) have caused enough problems that I hope we're at least starting to learn the lessons of the past. Using the same syntax for distinct concepts is sure to cause consternation.

*Specifically uniquely determined by the generic parameters of their type.

### A new propsal: `type Foo: Bar = _`
Though it's helpful to point out flaws with *existing* proposals, without a good alternative, it doesn't help the discussion progress. Fortunately, I have a new syntax that I think fits the intended use cases _better_ than `impl Trait`, while also being consistent with existing syntax.

This proposal promotes `type Foo: Bar = _` as a type alias for some type `T: Bar`, such that the compiler must infer the type `T`. The type `T` is unique — that is, there must be only one possible type `T` that satisfies the restriction (just like return-position `impl Trait`). This syntax is unambiguous and fits in with existing usage of `_` in turbofish (e.g. `.collect::<Vec<_>>()`) and type annotations (e.g. `let x: _ = 0u8;`). To reiterate: the `_` always represents a single type, and this is true in existing Rust syntax. So it's entirely consistent with our goal.

How is it different to existing to the previous proposals? Unlike the others, it's not _actually_ a syntax for existential types — it's a syntax for type inference. But semantically, it can be used anywhere return-position `impl Trait` type aliases could be, so it's functionally equivalent for these purposes. (I argue that we're not actually looking for a syntax for existential types — they're just what were leapt to as a solution — but they aren't the only one.)

On top of this, the underlying (inferred type) is transparent. This means that we *can* make assumptions about which type `Foo` is in `type Foo: Bar = _`. This is actually a good thing: you don't lose any information, which makes doing things like implementing traits for the type straightforward and calling inherent methods on the type. If you happen to what an opaque definition, you can wrap it up in a newtype — i.e. a single-field `struct`.

#### Unbounded existential types
As the compiler is inferring the type `_` in the proposal, it's often going to be possible to infer the type even without a trait bound. Therefore, we could even consider the unbounded syntax `type Foo = _` (which leads in naturally to declaring trivial bounds on trait aliases).

In theory, you could allow `_` anywhere in a type alias (the semantics of which was one point of contention with the previous proposals). `type Foo: Bar = (_, _)` would require its two components to be inferred separately.

However, I think these features are best left for a later proposal, to avoid getting caught up in even more tangential discussion now. Let's leave it as a promising note for the future.

In conclusion, I think this is an intuitive and consistent solution to the problem with existential type declarations that avoids a lot of the problems with the previously proposed syntaxes. This involved quite a lot of setting up — but it's an intricate issue — so I absolutely think it's best to be as unambiguous as we can when we're thinking about these concepts. Let's see where we can get with this fresh perspective.

### One last word
Though this formulation is much cleaner from a syntactic and theoretical perspective, there are still some loose ends. Using this type alias, we have a disconnect between the `impl Trait` syntax and the type alias syntax. Is there any way we can reconcile them? In fact, there might very well be a solution that has the benefits of both worlds... but I think we'll leave that for next time. I think we have enough food for thought for now.
