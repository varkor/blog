---
layout: post
title:  "A new perspective on impl Trait"
date:   2018-07-04 22:43:00 +0100
---
In my last post, [I described how to precisely formulate `impl Trait` as a form of existential type](https://varkor.github.io/blog/2018/07/03/existential-types-in-rust.html). This explanation hopefully cleared up some of confusion that I had seen around viewing `impl Trait` as an existential type.

However, a big part of the problem stemmed from the type theoretic terminology. Specifically, the term "existential type" was often thrown around, without anyone *really* explaining what it meant. (And while it may have been clear to some, it was definitely, understandably, murky for many.)

Really, even with the model in the post, there are some unsatisfactory consequences. Specifically:
- We're still using type theoretic language, which I think it's safe to say is not familiar territory for most Rust programmers.
- The existential type described by `impl Trait` is different depending on APIT (argument-position `impl Trait`) versus RPIT (return-position `impl Trait`).
- An arguably intuitive syntax `type Foo = impl Bar` is [highly inadvisable, due to ambiguity](https://varkor.github.io/blog/2018/07/03/existential-types-in-rust.html#proposal-1-type-foo--impl-bar).

It would be great if we could find a way to describe `impl Trait` in a way that avoids these problems (*without* introducing worse ones). As you've probably surmised by the existence of this post at all, there might be a way we can achieve this.

(Most of the credit for kicking off this discussion goes to [@rpjohnst](https://github.com/rpjohnst).)

## `impl Trait` from the perspective of type inference
Though the term "existential type" has been thrown around a lot, including in the `impl Trait` RFCs and official messages (e.g. blog posts), the semantics of `impl Trait` as they're currently implemented (in function signatures) *are not* uniquely satisfied by existential types. There's another consistent interpretation, which I think was obscured by the prevalence of theoretic terminology.

To do this, we need to consider the `_` symbol. When used in place of type, `_` stands for an inferred type variable. For example, in:
```rust
let x: _ = vec![1u8, 2u8, 3u8].into_iter().map(|n| [n]);
```
The `_` here stands for `std::iter::Map<std::vec::IntoIter<u8>, [closure@...]>`. (Of course, in this case, you can simply omit the type ascription, but that's arguably simply syntactic sugar.) In the following code:
```rust
let y = vec![1u8, 2u8, 3u8].into_iter().collect::<Vec<_>>();
```
The `_` here stands for `u8`.

Wherever you use `_`, you have to provide enough information for it to be inferred uniquely (that is: there must be exactly one choice for which type to infer). In theory, you could always replace `_` with the name of the type that's being inferred — because there's only one type it could be (though in practice, some types are not directly nameable, like closure types).

Let's consider what we would be possible if we could add a trait bound to `_`. I'm going to use the made-up syntax `_: Trait` (expressly not one I'm proposing) for now. Consider the following function signatures:
```rust
fn foo() -> (_: Trait);
fn bar((_: Trait)) -> (); // (*)
```
*The last syntax is slightly confusing because `:` is used both for type ascription and trait bounds.
What would the semantics of these functions be? Well, `foo` has a return type that is inferred, and `bar` has an argument whose type is inferred. `foo` can't return multiple types (for the same generic arguments, anyway), because inference must result in a unique type. Likewise, `bar`'s type is inferred from the argument passed in. (If you're not quite happy with this statement yet, go with it for now: we're going to get back to the precise semantics soon.)

You've probably noticed something about this (the heading *might* give it away). Our definitions of `foo` and `bar` are very similar to these definitions:
```rust
fn foo() -> impl Trait;
fn bar(impl Trait) -> ();
```

That is, `impl Trait` acts very similarly to type inference. In fact, under this interpretation, APIT and RPIT have *the same* semantics. This is something that the previous approaches (involving existential types) just haven't been able to do. And suddenly, once we have this interpretation, we open up new possibilities.
- We can finally stop talking about existential types completely (well, apart from our friend `dyn Trait`, but that's another story).
- We get back the syntax `type Foo = impl Bar`!, Now `impl Bar` has a single meaning, so we don't have to worry about which variant we're talking about.

There's a subtle but important point to make here, which involves transparency. Types inferred using `_` are always transparent (this is true for all uses of `_` as type inference in Rust today). This means that our hypothetical `_: Trait` syntax is *transparent* — you can observe exactly which type has been inferred. However, `impl Trait` is known to be opaque.

This isn't actually a problem. In this model, we define `impl Trait` thus:
- `impl Trait` is an inferred type with a trait bound, `Trait`,
- such that the bound and auto traits are observable,
- but the inferred type is opaque.

Note that the last two points are already part of the definition of `impl Trait` — it's just the first point that's different, switching from (varying) existential types to inferred types.

### Function semantics
To clarify exactly what's going on with the above examples, let me spell out exactly what I mean. We can view a `fn` definition as declaring a type implementing `Fn`, which is parameterised by its argument types and that has an associated `Output` type corresponding to the return type.
```rust
fn foo(A, B, C) -> D;
// desugars to...
impl Fn<(A, B, C)> for {foo} {
	type Output = D;
	/* ... */
}
```
We can now consider this desugaring from the perspective of an inferred type:
```rust
fn foo(impl Bar) -> impl Baz;
// desugars to...
impl Fn<(impl Bar,)> for {foo} {
	type Output = impl Baz;
	/* ... */
}
```
This has precisely the semantics we expect for `impl Trait`: in particular, the types of the arguments will always be inferred, whereas the return type, being an associated type, is fixed.

(Technically, because we're implicitly introducing generic parameters here, we're actually performing a sort of "ML-style let-polymorphism", which permits the inference of universal quantifiers.)

## The situation
`impl Trait` is an established (stabilised) part of the language. This means whatever model we pick to interpret `impl Trait` must be consistent with the existing usage in the language — specifically, function signatures.

Both the existential type model described in the previous post, and the type inference model described in this one, are consistent with the existing usage. However, adding an additional feature is likely to force a specific model. We should be very intentional about precisely which model offers more advantages (in terms of usage and explainability), rather than making an instinctive reaction (either way). I think it's more important at this stage to establish the precise model, *before* adding more features (such as `impl Trait` type aliases).

## Downsides
There's one obvious drawback to this approach, but I think it's significantly better than those surrounding the `impl Trait`-as-an-existential viewpoint.

### Referential transparency
In a pure language, you can always substitute the value of an expression with the expression itself. This means that if you have a variable `let x = y;`, you can always use `x` or `y` interchangeably. This obviously isn't true in a language with side-effects like Rust. However, right now, it is true for type aliases: if you have an alias `type Foo = Bar`, then you can always replace `Foo` with `Bar` and `Bar` with `Foo` anywhere in your code.

Consider `type Foo = impl Trait`. If you see `impl Trait` anywhere else in your code, you cannot replace it with `Foo`, because `Foo` is potentially more restricted (if used in more than one location). Likewise, you cannot replace any occurrence of `Foo` with `impl Trait`, because it'll be potentially less restricted. Though these changes aren't visible in the type system (i.e. your program will still type-check, assuming it did before), it could affect behaviour if you have specialising `impl`s, as different types will be inferred.

## Future opportunities
`impl Trait` is opaque, but following the proposal in the last post, there's also a sensible syntax suggestion for transparent type aliases, namely: `type Foo: Bar = _`. This flexibility: consistently disambiguating between opaqueness and transparency is a strength, though I think the transparency type alias would be better suited for an RFC after that for `type Foo = impl Bar`. Exercising caution is never a bad thing in language design.

In conclusion, I think that by viewing `impl Trait` in this alternative way, completely distinct from existential types, we're able to formulate a much more satisfactory semantics, which leads into more intuitive syntax for features like type aliases.