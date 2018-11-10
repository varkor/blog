---
layout: post
title:  "Monadic do notation in Rust: Part I"
date:   2018-11-10 18:23:56 +0000
latex:  true
---
Following [last time](https://varkor.github.io/blog/2018/08/28/feasible-functors-in-rust.html), where we saw that, given parameterision over traits (rather than just types), we could implement functors and monads in Rust that supported existing "monad-like" traits like `Iterator` and `Future`, I thought it would be interesting to tackle another one of the [arguments against monads in Rust](https://twitter.com/withoutboats/status/1027702531361857536).

I view this one more as an aside than a central point in the argument, but it's worth addressing anyway (it's also easier to solve than the general case of [higher-kinded types (HKTs)](https://en.wikipedia.org/wiki/Kind_(type_theory)), which makes it attractive to cherry-pick).

What's the point in question?

> Rust's imperative control flow statements like `return` and `break` inside of do notation also doesn't make sense, because we do not have TCP preserving closures.

[(Source.)](https://twitter.com/withoutboats/status/1027702535707090944)

Let me clarify the objection that's being made here.

"`do` notation" refers to [Haskell's `do` notation](https://en.wikibooks.org/wiki/Haskell/do_notation), which is a special syntax for manipulating monads without necessitating long sequences of nested functions (we'll see an example of this shortly).

"TCP" is an obscure initialism[^1], but people seem to use it in this sense: a language follows "TCP" if any expression `expr` is equivalent to `(|| expr)()`[^2]. In general in Rust this is true, but it breaks down when `expr` contains control flow. Take the following program:

[^1]: It stands for "Tennent's Correspondence Principle", if you're wondering, but it's not a standard term in programming language design. As far as I can tell was first mentioned in [this old Rust internals post](https://internals.rust-lang.org/t/pre-rfc-allow-return-continue-and-break-to-affect-the-captured-environment/4791). [This Stack Exchange](https://softwareengineering.stackexchange.com/a/120409) post gives a bit more context.

[^2]: This is simply $$\eta$$-equivalence for functions whose domain is the unit type `()`. (In general, $$\eta$$-equivalence for functions doesn't hold for call-by-value languages with effects, but it's *plausible* in this special case it could hold, as `()` has no effect.)

```rust
// This function returns `5`.
fn early_return() -> u8 {
    return 5;
    0
}
```

If we enclose `return 5;` in a closure, we get a different result:

```rust
// This function returns `0`.
fn abstracted_early_return() -> u8 {
    (|| return 5)();
    0
}
```

Why is this a problem? We'll need to take a look at an example of `do` notation to make this clearer. I'm going to use a hypothetical syntax that, while not being too pretty, should at least be functional for our examples.

(Rather than write out the fully explicit syntax as in the [last post](https://varkor.github.io/blog/2018/08/28/feasible-functors-in-rust.html), I'm going to assume the function traits are implicit and use the shorthand `impl (impl Trait)` for a type that implements a trait that implements a trait-on-traits. Obviously, if higher-order traits were to exist in Rust, we'd need a nicer syntax for them.)

Take the following snippet:

```rust
// `monadA` has type `impl (impl Monad<A>)`.
// `monadB` has type `impl (impl Monad<B>)`.
let monadAB = do! {
    // The type annotations aren't necessary,
    // but hopefully make things clearer.
    //
    // `let!` takes a monad and binds (using
    // `Monad::bind`) its "inner value" to a
    // new variable.
    let! a: A = monadA;
    let! b: B = monadB;
    // `return!` takes a value and wraps it in
    // a monad (using `Monad::unit`).
    return! (a, b);
};
// `monadAB` has type `impl (impl Monad<(A, B)>)`.
```

This is intuitively equivalent to:

```rust
monadA.bind(|a| monadB.bind(|b| Monad::unit((a, b))))
```

(If you're familiar with Haskell, `let a! = monadA;` corresponds to `a <- monadA` and `return! expr;` corresponds to `return expr`.)

You can see that the `do` notation version is a lot more readable. With more complex sequences of `bind`s and `unit`s, the difference becomes even more pronounced.

However, there's a problem with desugaring this in the obvious way shown above. We'd like to be able enclose any normal Rust block inside a `do!`. However, let's take a look at what happens when we introduce control flow.

```rust
do! {
    // This loop is a little pointless, but
    // it gives us something to break from.
    loop {
        let! a: A = monadA;
        break;
        let! b: B = monadB;
        println!("{} {}", a, b);
    }
}
```

This naïvely desugars to:

```rust
loop {
    monadA.bind(|a| {
        break; // Uh oh...
        monadB.bind(|b| {
            println!("{} {}", a, b);
        })
    })
}
```

There's an obvious problem: we expect `break` to break out of the `loop`, but since it's now inside a closure, it's not going to work (in fact, it won't even compile). This is the problem [withoutboats](https://github.com/withoutboats) is referring to when they say that `return` and `break` don't make sense in `do` notation.

We could simply forbid control flow expressions in `do!`, but this is very much an artificial (and to the user, a seemingly-arbitrary) solution and limits the general applicability and usefulness of `do` notation in Rust. Fortunately, however, there's a solution.

## A better desugaring for `do!`
We'll use a less specific example for the proposed desugaring so that it's slightly clearer.

Take the `do` notation below:
```rust
do! {
    expr1;
    let! a = expr2;
    expr3;
    let! b = expr4;
    expr5;
    expr6(a, b);
    return! expr7;
}
```

As a reminder, this is the naïve desugaring:

```rust
expr1;
expr2.bind(|a| {
    expr3;
    expr4.bind(|b| {
        expr5;
        expr6(a, b);
        Monad::unit(expr7)
    })
})
```

As we saw, this isn't good enough, so instead, we're going to desugar it like *this*[^3]:

[^3]: `surface!` and `bubble!` here aren't actually macros: in actuality, we would probably define the desugaring in a single step, which would render them unnecessary, but they help illustrate what's going on.

```rust
surface! {
    expr1;
    bubble! expr2.bind(|a| {
        expr3;
        bubble! expr4.bind(|b| {
            expr5;
            expr6(a, b);
            Monat::unit(expr7)
        })
    })
}
```

where `bubble! expr` desugars to:

```rust
match expr {
    ControlFlow::Return(_) => return expr,
    ControlFlow::Break(Some(_)) => break expr,
    ControlFlow::Break(None) => break,
    ControlFlow::Continue => continue,
    ControlFlow::Value(_) => expr,
    // We'd also want `Yield` here eventually,
    // but that comes with its own problems,
    // which is a story for another time.
}
```

We've got a couple of options for `surface!`, depending on whether or not we want control flow to be able to bubble up out of `do!` (for instance, whether a `return` inside `do!` will return out of the function enclosing the `do!` or not).

If we do, `surface! block` desugars to:

```rust
match (|| block)() {
    ControlFlow::Return(t) => return t,
    ControlFlow::Break(Some(t)) => break t,
    ControlFlow::Break(None) => break,
    ControlFlow::Continue => continue,
    ControlFlow::Value(t) => t,
}
```

If we want to forbid control flow at the top level, we need some custom error handling, but it's straightforward, technically, to implement.

The `enum` `ControlFlow` is defined:

```rust
enum ControlFlow<T> {
    Return(T),
    Break(Option<T>),
    Continue,
    // `Value` simply means we're forwarding
    // the value without effecting any control
    // flow.
    Value(T),
}
```

Essentially, `ControlFlow` reifies the control flow. By capturing the control flow inside a closure and forwarding it on immediately outside the closure, we can pretend that the control flow escapes the closure directly, which is precisely what we want for `do` notation.

## Final notes

As a serious proposal for a `do` notation desugaring in Rust, this has its flaws. Simulating the control flow in this way increases branching and it is thus likely not to have ideal performance. If we actually did want some form of `do` notation, we'd probably prefer to handle control flow more directly in the compiler, rather than simulating it using `ControlFlow`. But hopefully this demonstrates that it's not *necessary* to have special handling to support monadic `do` in Rust and at least it's not an issue with control flow that makes `do` notation implausible.

The issue with control flow in `do` notation is not the only one that was raised (see [this tweet](https://twitter.com/withoutboats/status/1027702532821463041) and [this one](https://twitter.com/withoutboats/status/1027702534331412482) for the others), so we don't have a full solution yet, but by tackling the difficulties one at a time we can get closer to a point at which we understand where the difficulty with these abstractions in Rust really lie.
