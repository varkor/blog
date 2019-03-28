---
layout: post
title:  "Idiomatic monads in Rust"
date:   2019-03-28 00:00:28 +0000
---
# A pragmatic new design for high-level abstractions
In this post, I'm going to describe a new approach to express monads in Rust. It is the most minimal design I have seen proposed and is, in my eyes, the first plausible design for such abstractions --- those commonly known as "higher-kinded types". This approach depends on a very minimal extension to Rust's type system. In particular, this approach avoids the need for either higher-kinded types (e.g. as in [this design](https://m4rw3r.github.io/rust-and-monad-trait)) or full abstraction over traits (e.g. ["traits for traits"](https://varkor.github.io/blog/2018/08/28/feasible-functors-in-rust.html)). Most of the design challenges are tackled directly using existing features.

However, without explaining *why* these design choices have been made, the final design can seem opaque. In particular, the design is significantly influenced by several quirks found in the Rust type system that will be unobvious to those who have not already come across them. To make this design accessible and hopefully also to explain why monads are hard(er than you might think) in Rust, I'm going to build up the definition gradually, explaining the motivation behind the choices.

I am going to be assuming some familiarity with [monads](https://en.wikipedia.org/wiki/Monad_(functional_programming)) (and to a certain extent, [`do` notation](https://en.wikibooks.org/wiki/Haskell/do_notation)). There are enough introductions to monads out there as it is.

The design here was influenced by conversations with [@Centril](https://github.com/Centril). Thanks also to [@Nemo157](https://github.com/Nemo157) for providing illuminating insights into subtleties with the behaviour of `yield`.

## Overview

There are several problems with a naïve design of monads (and similar abstractions) in Rust. The most prominent are the following.

- Monads are naturally structures at the level of type constructors, not types. Extending Rust to support full higher-kinded types raises many significant design questions. There are arguments that such an extension is simply infeasible.
- Monads abstract over function types, but Rust has many types of function (notably any type that implements `Fn`, `FnMut` or `FnOnce`)[^linear-monads].
- As traits are the natural abstraction mechanism in Rust, many structures that appear monadic (such as iterators) are not, strictly speaking, monads at the type level. For instance, calling `map` on *any* iterator always produces an `iter::Map`, rather than preserving the original type constructor[^external-iterator].
- Monadic structure appears at two levels in Rust: at the type level, such as with `Option`; and at the trait level, such as with `Iterator`. Abstracting over both elegantly is tricky.
- A naïve implementation of `do` notation would be surprisingly limited in Rust, as control-flow is captured by closures and would thus be useless in `bind` desugarings.

[^linear-monads]: A similar problem arises in Haskell when [linear function types are introduced](https://m0ar.github.io/safe-streaming/2017/07/20/homegrown-linear-monads.html).

[^external-iterator]: This is known as the *external iterator* pattern and is done for performance reasons.

As such, any design that puports to facilitate monads must provide solutions for each of these problems.

Let's see how we can do so.

## Idea

We'll see how this works in detail below, but here's a summary of the idea behind this design, to set the scene. Traditionally higher-kinded types cannot be defined naïvely, because we cannot abstract over type constructors in Rust. However, we can define their structure and properties _pointwise_ on each type that is an instance (that is, instead of saying something like "`Option` is a `Monad`", we instead say "`Option<A>` is a `Monad<A>` for all types `A`"). This is still not straightforward, however, because by defining a higher-kinded type in terms of each of its instantiations, we lose some higher level typing information. To remedy this, we employ generic associated types and generic associated traits (the one additional language feature) to restore the lost information. This way, `Monad` may be defined as a normal trait.

This is all a bit hand-wavy without seeing it in action, though, so we'll leave the theory there and get into the details.

## Functors

[Famously](http://james-iry.blogspot.com/2009/05/brief-incomplete-and-mostly-wrong.html), monads are functors. That means that before we can give a full definition of a monad, we have to give a definition of a functor.

For illustrative purposes, a definition of a functor in Haskell is as follows. The meaning should hopefully be clear even if you're not familiar with Haskell's syntax (just read `class` as `trait` and lowercase letters as uppercase ones).

```haskell
class Functor f where
    map :: f a -> (a -> b) -> f b
```

If we try to implement this in Rust, we immediately run into *several* problems.

(You may object to some, or all, of these definitions. That's a perfectly reasonable reaction. There's really no natural naïve definition.)

#### Note on syntax

I'm going to start by using `impl Trait` in both argument- and return-position, because I think it indicates intent more clearly. Later, I'll demonstrate why these are insufficient, but for now I want to prioritise readability.

### Attempt 1: Naïve definition

The `Functor` type classes in Haskell (what we'd expect to be the equivalent of a hypothetical `Functor` trait in Rust) is parameterised by a single type variable. So a first attempt at an analogous definition in Rust might look something like this.

```rust
trait Functor<A> {
    fn map<B>(Self, fn(A) -> B) -> /* ??? */;
}
```

There's clearly a problem here. We can't even define the return type of `map`. The problem is that we have no way to get a handle on the type constructor that we're implementing `Functor` for. `Self` is the type we're implementing `Functor` for, specifically *when the type parameter is `A`*. For example, if we're implementing `Functor` for `Option`, then `Self` would be `Option<A>` and we have no way to declare the type `Option<B>`.

The key insight here is that what we'd really like is a higher-kinded `Self`. That is, we'd like to write something like the following.

```rust
trait Functor<A> {
    fn map<B>(Self<A>, fn(A) -> B) -> Self<B>;
}
```

This just doesn't work with `Self` as-is, but the idea is promising. What we're going to do is create a "higher-kinded" version of `Self` ourselves, using generic associated types.

### Attempt 2: Generic associated `Self`

```rust
trait Functor<A> {
    type HigherSelf<T>: Functor<T>;

    fn map<B>(Self, fn(A) -> B) -> Self::HigherSelf<B>;
}
```

With a generic associated type, we can actually define a signature for `map` that looks like something we might expect. (Think of `Self` as being the same as `HigherSelf<A>`.) We might almost be able to convince ourselves that this is a reasonable definition. Unfortunately, functions aren't all that simple in Rust (as you'll know if you've been using Rust for a little time).

#### Aside on identity

Here, we see a generic type `HigherSelf<T>` that we are pretending is the type we're implementing a trait for. In the definition above, though, the type is completely generic; there's no reason to believe that `Self` and `HigherSelf<T>` are related in any way. We can address this with [type equality constraints](https://github.com/rust-lang/rust/issues/20041), like in the following.

```rust
type HigherSelf<T> where Self = HigherSelf<A>;
```

This ensures our notion of "higher-kinded `Self`" is somewhat well-behaved. This kind of condition gets more and more complex to declare as we progress through the design, so I've opted to leave them out here. In a real implementation, one may or may not want to enforce this property: having these equality relations is truer to the original concept, but in practice probably has little additional benefit.

### A family of functions

Rust has several notions of "function", which is another reason the definition of `Functor` isn't quite so straightforward as in Haskell[^linear-monads]. Namely, in Rust we have the following.

- Unique types for each function or closure. For example the function `fn foo() {}` has the unique type `fn() {foo}`. (This type is unnameable in Rust, but may be printed in diagnostic messages.) Two functions that have the same signatures will have different types. [There are historical reasons for this.](https://github.com/rust-lang/rfcs/blob/master/text/0401-coercions.md)
- Function pointers, such as `fn(bool) -> (u8, u8)`. Unique function types can be coerced to function pointers. So can unique closure types, as long as they *do not capture variables from their environment*.
- Three traits for abstracting over functions: `Fn`, `FnMut` and `FnOnce`. These are automatically implemented by unique function types and unique closure types (the specific traits the closure type will implement depends on how it captures from the environment).

The `Fn*` traits are the correct way to abstract over functions in Rust: there's no single function type constructor `(->)`. Thus, we need to generalise our definition a little further.

### Attempt 3. Generalised mapping function

This is where we need a new language feature. The only facility we require for this design that is not currently an accepted language feature in Rust is the concept of (generic) associated traits. If you're familiar with associated types and associated consts, the idea will be obvious. The reason we need associated traits is to abstract over the three `Fn*` traits. Instead of hard-coding a specific kind of function in our functor definition, we'll allow it to be specified in the implementation of `Functor`.

```rust
trait Functor<A> {
    type HigherSelf<T>: Functor<T>;

    trait MapFn<T, U>;

    fn map<B>(Self, impl Self::MapFn<A, B>) -> Self::HigherSelf<B>;
}
```

In practice, we would expect `MapFn<A, B>` to be instantiated to one of `Fn(A) -> B`, `FnMut(A) -> B` or `FnOnce(A) -> B`, and we could enforce this, but there's no real need to do so. By leaving it generic, we create a more general abstraction[^enriched-functor] (that's no less useful).

[^enriched-functor]: From the perspective of category theory, this corresponds to defining a [functor from an enriched category](https://en.wikipedia.org/wiki/Enriched_category#Enriched_functors).

So, now we're done, right? This seems like a perfectly good definition of a functor.

Well... actually, not quite yet. We can see this isn't quite general enough if we look at a trait that we expect to be functorial. The `map` method on `Iterator<Item = A>` has the following signature.

```rust
fn map<B, F: FnMut(A) -> B>(Self, F) -> iter::Map<Self, F>;
```

Notice how the return type has an extra parameter in it: the (unique) type of the mapping function `F`. (The reasons for this are to facilitate a design pattern called *external iteration*, but the motivation is really tangential here. The fact that we want `Iterator` to be functorial means we need to abstract over this design pattern.)

In general, a function's return type could capture *all* of the input type parameters that are passed to the function. While it may not look like it in our signature, `impl Trait` in argument-position corresponds to a generic type argument. We need to take account of this detail: namely, we need to add this additional type parameter to the generic return type (that is, `HigherSelf`).

### Attempt 4. Capturing generic arguments

Now that the return type also needs to capture the mapping function, it doesn't make so much sense to call it `HigherSelf`: it still corresponds in some sense to `Self`, but at the same time, it's been specialised to the `map` function signature. In general, if our traits have multiple methods, we'll need different generic associated traits for each, so it makes sense to use a different name. I'll call it `Map` for simplicity (not to be confused with the identically-named `iter::Map`... sorry).

```rust
trait Functor<A> {
    type Map<T, F>: Functor<T>;

    trait MapFn<T, U>;

    fn map<B, F: Self::MapFn<A, B>>(Self, F) -> Self::Map<B, F>;
}
```

With this last definition, we really are done. It's taken several iterations, but this definition of `Functor` is general enough to capture the examples of functors that we're interested in.

### Implementing `Functor`

We've defined the trait; the next thing to do is to implement it. After deriving a correct definition, making use of it presents no problems.

I'm just going to give two definitions: for a functorial type and a functorial trait, to demonstrate the flexibility of our definition. More examples can be found at the end of this post.

```rust
// Implementing `Functor` for a type.
impl<A> Functor<A> for Option<A> {
    type Map<T, F> = Option<T>;

    trait MapFn<T, U> = FnOnce(T) -> U;

    fn map<B, F: FnOnce(A) -> B>(self, f: F) -> Option<B> {
        self.map(f)
    }
}

// Implementing `Functor` for a trait.
impl<A, I: Iterator<Item = A>> Functor<A> for I {
    type Map<T, F> = iter::Map<T, F>;

    trait MapFn<T, U> = FnMut(T) -> U;

    fn map<B, F: FnMut(A) -> B>(self, f: F) -> iter::Map<B, F> {
        self.map(f)
    }
}
```

## Monads

Many of the difficulties in defining `Monad` are those we already encountered when figuring out how to implement `Functor`: the hardest parts are behind us. However, there is one subtlety in particular in how we define `bind`, which is a slightly more complex situation than `map`.

Let's look at the definition in Haskell again first, which I think clearly presents the signature we expect. (What I call `unit` and `bind` here are usually called `return` and `>>=` in Haskell.)

```haskell
class Monad m where
    unit :: a -> m a
    bind :: m a -> (a -> m b) -> m b
```

This time, I'm going to start with the complete definition in Rust, as it should be mostly familiar, and explain the parts that are new.

```rust
trait Monad<A>: Functor<A> {
    trait SelfTrait<T>;

    // Unit
    type Unit<T>: Monad<T> + SelfTrait<T>;

    fn unit(A) -> Unit<A>;

    // Bind
    type Bind<T, F>: Monad<T> + SelfTrait<T>;

    trait BindFn<T, U>;

    fn bind<B, MB: Self::SelfTrait<B>, F: Self::BindFn<A, MB>>(Self, F) -> Self::Bind<B, F>;
}
```

The first thing that you'll notice is that we have a new associated trait, `SelfTrait`. I'm going to come back to what role this plays shortly: you can ignore it for now.

The `unit` function is straightforwardly defined using the same reasoning as `Functor::map`. In the same way, we need to define a generic associated type that functions as its return type. (Here, there's an additional `SelfTrait` bound on `Unit`, but that's not important to understand yet[^self-trait-in-functor].)

[^self-trait-in-functor]: We could have added a `SelfTrait` to `Functor` too, and imposed a similar bound on `Functor::Map`, which would have made the definition stronger. However, I thought it clearer to leave it out until it was strictly necessary. That is, the weaker definition of `Functor` defined above is sufficient to implement the types we expect to be functors, albeit while  allowing us to implement `Functor` for some types we probably don't expect to be functors.

In general, though, for each function we define that uses `Self` in a higher-kinded fashion, we need to use a generic associated type just like for `map`. Such as is the case with `bind`. Here also, we want to permit the binding function type to vary, so we use an associated trait, `BindFn`, representing the kind of binding function, just like before. We could have reused the `MapFn` trait: often `map` and `bind` will take the same kind of functions, but this isn't always true, so for maximum generality we need another trait.

However, the definition of `bind` is a little different than `map`. Recall that the binding function has a signature of `A -> M<B>` (contrast to the mapping function's signature `A -> B`). This means we have to take a function that returns a monad (specifically, the current one) parameterised over `B`. However, the specific type of this monad may not be fixed. For example, consider the monad `Iterator`. The `bind` operation for the `Iterator` trait corresponds to the `flat_map` method. We should be able to return *any* type that implements `Iterator` from the flat-mapping function we pass to `flat_map`. It's probably easiest to demonstrate this with an example.

```rust
let xs = [1, 2, 3];

// Here, the flat-mapping function returns `iter::Take`.
let ys: Vec<_> = xs.iter().flat_map(|x| iter::repeat(x).take(2)).collect();
// `ys` contains `[1, 1, 2, 2, 3, 3]`.

// Here, the flat-mapping function returns `option::IntoIter<_>`.
let zs: Vec<_> = xs.iter().flat_map(|x| Some(x).into_iter()).collect();
// `zs` contains `[1, 2, 3]`.
```

Clearly, the return type of the function we pass to `flat_map` (and, correspondingly, to `bind`) can vary from call to call. Therefore, we want to simply ensure it returns *some* type that implements our monad. This is the role `SelfTrait` plays. Just like `Map` and `Bind` act like "higher-kinded `Self` types", `SelfTrait` acts like a "higher-kinded `Self` *trait*". (If this still doesn't click just yet, wait till you see the examples, which should make things clearer.)

It's worth noting that we only need to define one `SelfTrait`, even if we make use of it in multiple functions (unlike associated types like `Map` and `Bind`, which had to be defined on a per-function basis). This is essentially because generic arguments can vary, but the return type is fixed (up to their generic parameters). (This also effectively falls out of the difference between argument-position and return-position `impl Trait`, albeit in a slightly disguised form.

#### Note on return-position `impl Trait` in traits

If we had return-position `impl Trait` in trait definitions, we could eliminate the need for our specialised `Map` and `Bind` types entirely using our associated `SelfTrait`. This makes for a much cleaner definition. Here's `Functor`.

```rust
trait Functor<A> {
    trait SelfTrait<T>;

    trait MapFn<T, U>;

    fn map<B, F: Self::MapFn<A, B>>(Self, F) -> impl Self::SelfTrait<B>;
}
```

This is technically just syntactic sugar, and it's not necessary for any of the definitions here, as we've seen. It does simplify things though (and is arguably closer to the higher-kinded viewpoint, as we're defining everything from a higher level of abstraction[^opaque-impl-trait]).

[^opaque-impl-trait]: Technically, using `impl Trait` is actually a little more restrictive than we intend, because `impl Trait` is opaque: the underlying type is not observable. What we'd really like here is some kind of [transparent analogue](https://varkor.github.io/blog/2018/07/04/a-new-perspective-on-impl-trait.html#impl-trait-from-the-perspective-of-type-inference).

### Identities

If you were following the previous definition closely, you may have spotted an apparent oversight. In particular, the associated `SelfTrait` only makes sense for monadic traits. If we want to implement `Monad` for a *type*, then there's no such trait! In the case of a monadic type, the binding function should return something of the (parameterised) type *precisely*, not just something that implements some trait.

As we saw, though, we do need this level of abstraction for monadic traits. Fortunately, there's a tidy way we can make `Monad` work just as nicely for monadic types. The key is a generic "identity trait". The identity trait on `T` will be implemented *solely* for `T`. No other type will be permitted to implement the identity trait (this is often known as a *sealed trait*). It's straightforward to define.

```rust
trait Id<T> {}
impl<T> Id<T> for T {}
```

Now, for example, `Option<bool>` (and no other type) implements `Id<Option<bool>>`. For monadic types, we can now simply use `Id` for `SelfTrait` and everything works as expected.

### Implementing `Monad`

```rust
// Implementing `Monad` for a type.
impl<A> Monad<A> for Option<A> {
    trait SelfTrait<T> = Id<Option<T>>;

    // Unit
    type Unit<T> = Option<T>;

    fn unit(a: A) -> Option<A> {
        Some(a)
    }

    // Bind
    type Bind<T, F> = Option<T>;

    trait BindFn<T, U> = FnOnce(T) -> U;

    fn bind<B, MB: Id<Option<B>>, F: FnOnce(A) -> MB>(self, f: F) -> Option<B> {
        self.and_then(f)
    }
}

// Implementing `Monad` for a trait.
impl<A, I: Iterator<Item = A>> Monad<A> for I {
    trait SelfTrait<T> = Iterator<Item = T>;

    // Unit
    type Unit<T> = iter::Once<T>;

    fn unit(a: A) -> iter::Once<A> {
        iter::once(a)
    }

    // Bind
    type Bind<T, F> = iter::FlatMap<T, F>;

    trait BindFn<T, U> = FnMut(T) -> U;

    fn bind<B, MB: Iterator<Item = B>, F: FnMut(A) -> B>(self, f: F) -> iter::FlatMap<B, F> {
        self.flat_map(f)
    }
}
```

Easy!

## Deriving functors and monads
In the examples above, though the implementations of `Functor` and `Monad` for types and iterators are straightforward, they do contain a lot of boilerplate. This is an unfortunate consequence of the lack of full higher-kinded types: we generally have to be much more explicit about the various types. This is likely to be true for any similar representation of higher-kinded types in Rust.

However, in practice, we could avoid this boilerplate in the majority of instances. This is made clear by observing how the entirety of the implementation definitions are determined by the `map`, `unit` and `bind` functions for `Functor` and `Monad` respectively. Often types and traits implementing these higher-kinded traits already define these functions (albeit with different names): we've already seen this with `Option` and `Iterator`, where we simply delegate to existing functions, and it's certainly the case for many of the functors and monads in the Rust standard library.

In these cases, we could use a custom `#[derive]` attribute to generate the boilerplate automatically for us in the background. With this feature, implementing `Functor` and `Monad` could be as simple as the following.

```rust
#[derive(Functor(map), Monad(Some, and_then))]
enum Option { ... }

#[derive(Functor(map), Monad(once, flat_map))]
trait Iterator { ... }
```

The message to take away here is that, though the definitions might seem intimidating at first, in all likelihood they would rarely need to be implemented directly by the user.

## Do notation
It would be remiss of me to talk about the feasibility of monads without even mentioning `do` notation. For those of you who are unfamiliar with `do` notation, it provides a convenient synactic sugar for working with monads without having to make do with deeply-nested `bind`s and `unit`s in Haskell. Monads are useful abstractions even without `do` notation, but there is definitely something to be said for a design that also encompasses a similar synactic sugar for monads in Rust.

The main argument against the feasibility of `do` notation in Rust is the difficulty with composing control flow expressions (such as `return` or `break`) with closures. This is because closures capture control-flow: we can't break out of a loop enclosing a closure within the closure itself, for instance.

```rust
loop {
    || break; // This isn't going to work.
}
```

This is a problem, because `do` notation is desugared in terms of nested `bind`s, which take closures as arguments. What we might expect to work intuitively in a naïve implementation of `do` would fail unexpectedly.

Let's look at a concrete example to see exactly where this fails.

```rust
// This is a useless loop, but it's illustrative.
loop {
    do {
        let x <- a;
        println!("x is {}.", x);
        break; // We've done what we came for: let's leave the loop.
    }
}
```

Desugaring `do` from a naïve perspective, we might expect this to be equivalent to the following.

```rust
loop {
    a.bind(|x| {
        println!("x is {}.", x);
        break; // Uh oh...
    });
}
```

Obviously, this doesn't work, because we're trying to break the outer loop from inside the `bind` closure. However, from the perspective of the `do` notation, it seems entirely reasonable.

There's a happy conclusion to this story. [I wrote a separate post on the topic](https://varkor.github.io/blog/2018/11/10/monadic-do-notation-in-rust-part-i.html) a while ago, describing how we can resolve this dichotomy. In essence, control flow and `do` notation are *not* mutually exclusive: you just need to be a little cleverer in your desugaring. We might desugar something like the following[^placement-new]:

[^placement-new]: To those of you who recognise the `<-` sigil as being experimental syntax for "placement new", just note that I'm using `<-` here for monadic bindings, reminiscent of the `do` notation from Haskell, *not* placement new.

```rust
let expr0 = do {
    expr1;
    let a <- expr2;
    expr3;
    let b <- expr4;
    expr5;
    expr6(a, b);
    expr7
};
```

into, intuitively, something of the form:

```rust
let expr0 = surface! {
    expr1;
    bubble! expr2.bind(move |a| {
        expr3;
        bubble! expr4.bind(move |b| {
            expr5;
            expr6(a, b);
            Monad::unit(expr7)
        })
    })
}
```

where, for the sake of illustration, `surface!` and `bubble!` are macros that perform propogation of control flow.

```rust
use ControlFlow::*;

// `bubble!(expr)` desugars to...
match expr {
    Return(_) => return expr,
    Break(Some(_)) => break expr,
    Break(None) => break,
    Continue => continue,
    Yield(Some(_)) => yield expr,
    Yield(None) => yield,
    Value(_) => expr,
}

// `surface!(expr)` desugars to...
match (|| expr)() {
    Return(t) => return t,
    Break(Some(t)) => break t,
    Break(None) => break,
    Continue => continue,
    Yield(Some(t)) => yield t,
    Yield(None) => yield,
    Value(t) => t,
}
```

In practice, this exact desugaring has some rough edges. For example, any variables referenced inside a `do {}` block are moved inside the `bind`ing closures, and are then unusable after the scope of the `do {}` block.

This is a flaw that could be addressed with built-in support from the compiler for `do` notation (for example, it could move those variables live at the end of the `do {}` block's scope to the enclosing scope). In practice, if we decided we wanted `do` notation in the language, we'd want to experiment with exactly how `do` should work, especially in its interaction with lifetimes. The main takeaway here is that `do` notation in Rust could make sense: even though by necessity we're working with closures behind the scenes, we can still recover the expressivity of the notation with some sleight of hand. (Whether `do` notation would be useful in practice is another question, which would be a question for a full feature proposal, rather than the exploration here.)

## Abstracting over monads
We're almost done. The last topic I want to cover is the question of further abstraction. There are really two questions that are natural to ask at this point.

1. How can we write abstractions involving monads?
2. How can we abstract *over* monads?

The first question is easily answered. As `Monad` is simply a trait, we can use it like any other. As a user of `Monad`, we don't need to worry about whether an implementer is a monadic type or a monadic trait: whatever form it takes, we know it conforms to the trait interface specified by `Monad`.

```rust
// A simple function making use of a monad.
fn double_inner<M: Monad<u64>>(m: M) -> M {
    m.bind(|x| M::unit(x * 2))
}
```

Practically, monads are now no more imposing than any other trait. All the complexity is hidden away in the definition (and to some extent, the trait implementation).

The second question requires a little more thought to answer. Functors and monads are, in one respect, simply the first rung on the ladder of higher-kinded types. Just as monads (as type constructors) abstract over types, we can also abstract over monads themselves. To give an example, one such abstraction is a [*monad transformer*](https://en.wikipedia.org/wiki/Monad_transformer).

Frankly, at this point, I'm not sure whether even higher-kinded types are possible to express in Rust. When I spent a little time playing around with potential definitions of monad transformers, I kept running into design problems: exactly the same tricks as for monads don't quite carry across, because we end up wanting to quantify over *all* traits in an implementation, rather than one trait per implementation. That said, I didn't try all that hard and it's possible there's a way to resolve the difficulties.

The main takeaway here is that, even though the designs here may not (obviously) generalise to *all* higher-kinded types, they generalise to a wide class of examples[^see-appendix] that could reasonably be considered sufficient for most of the abstraction users generally want to express in Rust. Even without full higher-kinded types, perhaps this is enough.

[^see-appendix]: See the appendix for a large range of examples of useful higher-kinded types that *are* expressible with the techniques advocated here.

## Summary
Let's take a step back and appreciate what we've achieved.

- We've defined an abstraction for functors and monads in Rust, two common higher-kinded types, using only one (conservative) hypothetical language feature: associated traits.
- We've demonstrated how it elegantly abstracts over both monadic types and monadic traits.
- We've shown how implementational boilerplate can be close to eliminated using `#[derive]` macros.
- We've illustrated how control flow is compatible with a natural `do` notation in Rust.

Many have expressed reservations about the feasibility of providing abstractions over higher-kinded types like monads in Rust (perhaps the most notable being [this thread](https://twitter.com/withoutboats/status/1027702531361857536), which I address directly in the appendix for completeness). The past proposals for monads in Rust that make use of advanced type system features such as full higher-kinded types do nothing to palliate these suspicions.

However, I think in this design, there is at last a plausible and reserved model for these abstractions. In particular, I think monads fall naturally out of a conservative extension of the language that is beneficial even without considering the potential for higher-kinded types.

At the very least, I hope that I have given some food for thought. We never needed higher-kinded types. My conclusion wherefore is this: monads *are* feasible in Rust.

## Appendix

### Accepting the challenge

> IN CONCLUSION: a design that works in a pure FP which lazily evaluates and boxes everything by default doesn't necessarily work in an eager imperative language with no runtime.

I haven't explicitly addressed each of the points in the aforementioned ["Monads and Rust" thread](https://twitter.com/withoutboats/status/1027702531361857536) in the main post, because I think it is self-evident that the design here is sufficient to capture `Monad` in Rust. However, I shall do so briefly here, to satisfy any suspicions of avoiding difficulties by omission.
- **Borrowing across `yield` in `do` notation**: there is a problem with pretending `yield` is simply the `bind` of the `Future` monad (despite both having similar effects), namely when borrowing is involved. (The only examples I've seen so far are rather involved, so I'm not going to reproduce them here.) That is: we can't *replace* the `yield` keyword with `<Future as Monad>::bind`. However, we can use `yield` just fine within `do` notation (using the same propagation of control flow from closures as `break` or `return`). So `yield` per se isn't a problem with `do` notation: we just can't abstract it away entirely.
- **Control flow in `do` notation**: as illustrated above, using a straightforward desugaring, we can recover full control flow within `do` notation (arguably solving [an open research question](https://twitter.com/withoutboats/status/1027702537217036290)).
- **External iterator patterns induces un-monadic signatures**: by abstracting over traits, rather than types, we can describe monadic signatures entirely idiomatically.
- **There are three kinds of function type constructor**: using associated traits, we can capture all three notions of function (and more).
- **Higher-kinded polymorphism is hard**: in this design, we completely avoid higher-kinded polymorphism (and full higher-kinded types in general).
- **Abstracting over monads would be unergonomic in Rust**: using `#[derive]`, implementing higher-kinded types can be made very minimal and ergonomic.

To conclude, there's no guarantee that a feature designed with a pure, lazy language in mind will work in an eager[^eagerness-irrelevant], imperative language. But sometimes it does; it just requires a little more care.

[^eagerness-irrelevant]: As we've seen, eagerness and laziness don't actually factor into the design problem at all.

### Additional examples

In this post I've focused on functors and monads for `Option` and `Iterator` in particular. The techniques here extend naturally to other popular examples of higher-kinded types and other monadic types and traits. For completeness, I'm going to give some additional examples, to demonstrate that the framework here really is general enough to encompass the traditional and desirable use cases. This appendix can be viewed as a reference more than a section worth reading in its own right.

#### `join`
Monads have another operation, known as `join`, or simply "multiplication", with the signature `m (m a) -> ma`. It's interderivable with `bind`, so I haven't included it in the definition above, but we could provide it as a default implementation[^default-limitations] on the `Monad` trait.

[^default-limitations]: Okay, I'm taking some liberties here. We *should* be able to define it automatically, but `impl Trait` in associated types is too limited at the moment. [There's an RFC](https://github.com/rust-lang/rfcs/pull/2515) (in the final stages at the time of writing) that should address this, though. You're still able to define `join` manually for each implementer of `Monad`, though: it's just a little less convenient.

```rust
trait Monad<A>: Functor<A> {
    // ...

    // Join
    type Join = Self::Bind<A, impl Self::BindFn<impl Self::SelfTrait<Self>, impl Self::SelfTrait<Self>>>;

    fn join<MA: Self::SelfTrait<Self>>(ma: MA) -> Self::Join {
        ma.bind(|a| a)
    }
}
```

In the case of `Iterator`, for instance, an explicit definition would look like this.

```rust
impl<A, I: Iterator<Item = A>> Monad<A> for I {
    // ...

    // Join
    type Join = iter::Flatten<Self>;

    fn join<MA: Iterator<Item = I>>(ma: MA) -> Self::Join {
        ma.flatten()
    }
}
```

#### Further `Monad` implementations

##### [`Future`](https://docs.rs/futures/0.1/futures/future/trait.Future.html)

```rust
impl<A, E, R: Future<Item = A, E>> Functor<A> for R {
    type Map<_, F> = future::Map<Self, F>;

    trait MapFn<T, U> = FnOnce(T) -> U;

    fn map<B, F: FnOnce(A) -> B>(self, f: F) -> future::Map<Self, F> {
        self.map(f)
    }
}

impl<A, E, R: Future<Item = A, E>> Monad<A> for R {
    trait SelfTrait<T> = Future<Item = T, E>;

    // Unit
    type Unit<T> = Ready<T>;

    fn unit(a: A) -> Ready<A> {
        future::ready(a)
    }

    // Bind
    type Bind<T, F> = AndThen<Self, T, F>;

    trait BindFn<T, U> = FnOnce(T) -> U;

    fn bind<B, MB: Future<Item = B, E>, F: FnOnce(A) -> B>(self, f: F) -> AndThen<B, F> {
        self.and_then(f)
    }
}
```

#### More higher-kinded types

(Some of the initial types are not higher-kinded, but are necessary for later definitions.)

#### Magma

```rust
trait Magma {
    fn mul(Self, Self) -> Self;
}
```

#### Semigroup

```rust
trait Semigroup: Magma {}
```

#### Monoid

```rust
trait Monoid: Semigroup {
    fn unit() -> Self;
}
```

#### [`Foldable`](http://hackage.haskell.org/package/base-4.11.1.0/docs/Data-Foldable.html)

```rust
trait Foldable<A> {
    trait FoldMapFn<T, U>;

    fn fold_map<M: Monad, F: FoldMapFn<A, M>>(Self, F) -> M;
}
```

##### [`Applicative`](http://hackage.haskell.org/package/base-4.11.1.0/docs/Control-Applicative.html)

```rust
trait Applicative<A>: Functor<A> {
    trait SelfTrait<T>;

    // Unit
    fn unit(A) -> Self;

    // Apply
    type Apply<T, F>: Applicative<T>;

    trait BindFn<T, U>;

    fn apply<B, F: BindFn<A, B>, T: SelfTrait<F>>(T, Self) -> Apply<B, T>;
}
```

### Other higher-kinded types

There are some higher-kinded types I suspect we need additional type system additions to support. For example, I think definining [`Traversable`](http://hackage.haskell.org/package/lens-4.17/docs/Control-Lens-Traversal.html) requires generic traits: i.e. traits as generic parameters. This is a similar, but distinct, feature from (generic) associated traits.

As another example, the similarly-named [`Traversal`](http://hackage.haskell.org/package/lens-4.17/docs/Control-Lens-Traversal) probably requires [higher-ranked types in trait bounds](https://github.com/rust-lang/rfcs/issues/1481).

These are both plausible language extensions that also do not introduce the full complexity required by general higher-ranked types. However, I do not dwell on them here: I think associated traits provide the most immediate benefit of these type system extensions (it's better to focus on one feature at a time).