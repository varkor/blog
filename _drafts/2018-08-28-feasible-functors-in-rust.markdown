---
layout: post
title:  "Feasible functors in Rust"
date:   2018-08-28 01:35:11 +0100
latex:  true
---
[withoutboats](http://github.com/withoutboats), one of the Rust language design team, recently posted a thread on [the infeasibility of monads](https://twitter.com/withoutboats/status/1027702531361857536) as a useful abstraction technique in Rust, as a response to the persistence of some (usually from outside the Rust community) in claiming that "Rust is doing things incorrectly" by developing specific solutions to problems, rather than using a general category theoretic framework for everything. The points demonstrate real difficulties with attempting to use a general framework for these problems and to me serves perfectly as a "the ball's in your court now" to anyone claiming Rust is ignoring theory and coming up with unnecessary solutions to solved problems: if you think Rust could use monadic abstractions, you have to be able to address these counterarguments.

Personally, I think the design decisions behind these features are sound and that a monadic abstraction for Rust is infeasible with the design constraints. However, I think the thread is interesting, academically, as a set of design challenges. These are a set of problems, without existing solutions, that could potentially have useful consequences if tackled.

In this post, I'm going to take a look at one of the assertions and see if there's any way we might address it. In doing so, I hope to show that a functorial abstraction for existing interfaces in Rust is not as impossible as some think.

[Note: it turned out that I wasn't the only one to be thinking about this problem recently. [Here's a very similar solution](https://gist.github.com/eddyb/5fcbc9ee442b6ce9c63bbf1ba54e70d6#gistcomment-2690193) to the one in this post.]

## The problem

> Okay, so **Monad can't abstract over Future**, but still let's have Monad.

([Source.](https://twitter.com/withoutboats/status/1027702544909455360) Emphasis mine.)

I want to challenge this statement, purely as an exercise, because abstraction is fun. I'm not going to present a complete solution, but a hint that these problems are not insurmountable.

Let's see what the arguments against `Future`'s monadicity were.

> Getting beyond that, this is assuming that "future" implements "monad," so if we could just add HKT to have a Monad trait, everything would be hunkydory. That's not true!

> the signature of >>= is `m a -> (a -> m b) -> m b`
the signature of Future::and_then is roughly `m a -> (a -> m b) -> AndThen (m a) b`

> That is, in order to reify the state machine of their control flow for optimization, both Future and Iterator return a new type from their >>= op, not "Self\<U>"

> Also, our functions are not a `->` type constructor; they come in 3 different flavors, and many of our monads use different ones (FnOnce vs FnMut vs Fn).

So what's the problem? Let's get our definitions straight first.

A [functor](https://en.wikipedia.org/wiki/Functor) $$F: \mathscr{C} \to \mathscr{D}$$ (for [categories](https://en.wikipedia.org/wiki/Category_(mathematics)) $$\mathscr{C}$$ and $$\mathscr{D}$$) is formed from two components:
- A map on types, sending a type $$A: \mathscr{C}$$ to a type $$F(A): \mathscr{D}$$.
- A map on functions, sending a function $$f: A \to B$$ to a function $$F(f): F(A) \to F(B)$$.

An example of a functor is the `List` type constructor along with `map` on lists, which:
- Takes a type `A` to the type `List(A)` of lists containing elements of type `A`.
- Takes a function `f: A -> B` and returns a function `List(f): List(A) -> List(B)`, which "maps" the function `f` over every element in its input.

Functors are often used for "mappable containers": some data structure that we can map a function over.

Intuitively, this seems to fit iterators quite well: in Rust, any type implementing `Iterator` has a `map` function, which looks just like the `map` in `List`.

However, if we take a look at the signature of `Iterator::map`, we can see something's wrong.

```rust
fn map<B, F>(self, f: F) -> Map<Self, F> where F: FnMut(T) -> B;
```

Let's write this out using type notation, because I think it's clearer. I'm using $$\mathcal{U}$$ for the category (universe) of types (or `Type` in the code examples), $$\text{Iterator}(A)$$ for some type implementing `Iterator<Item = A>` and $$\text{Map}$$ for the type constructor `iter::Map`. The function `Iterator::map` then has a signature that looks something like this:

$$\text{map}: \prod_{A, B: \mathcal{U}} \prod_{F: A \to B} \text{Iterator}(A) \times F \to \text{Map}(\text{Iterator}(A), F)$$

Whereas for a functor, we want something more like:

$$\text{map}: \prod_{A, B: \mathcal{U}} \prod_{F: A \to B} \text{Iterator}(A) \times F \to \text{Iterator}(B)$$

Notice how the return type is different: for a functor, we want the return type to be over the same family of types as the input, but for $$B$$ rather than $$A$$. However, regardless of what type `Iterator::map` is called on, it always returns an `iter::Map`: it's not as parameteric as the functorial map.

This is the fundamental problem: we want to express the type of a functor map generically, whereas with Rust it's always specialised to some specific data type (such as an external iterator like `iter::Map`). We just don't have a functor. Let's see how we can address this.

## What makes Rust special?
Why is this a problem in Rust but not other, functional languages like Haskell? It's primarily due to optimisation concerns. Rust makes performance a high priority: you should be able to safely use its high-level abstractions with little to (ideally) no cost. Naïve iterators (when chained together) are very inefficient, continually constructing and then deconstructing data structures. Rust's strategy ensures that even long chains of iterator methods are performant (this is known as internal versus external iteration).

Because of this design pattern (among others), Rust doesn't put so much of an emphasis on abstraction through types, instead preferring to use traits as the primary form of abstraction.

Thus, when considering this problem, the mistake is to get caught up in the types. There are several strategies for approaching this problem: for example, one could try to consider how we can convert from an `iter::Map` to another, arbitrary iterator, so that we can get the type we started with back (and hence restore the functorial nature of `Iterator::map` by composing it with the conversion function). This is ultimately an unilluminating rabbit-hole to go down, as it doesn't consider take the semantics of `Iterator` at all. Fundamentally, when you have an iteration, the underlying types (such as `iter::Map`, `iter::Once` or `iter::Chain`) are unimportant: it's the iteration itself that's key (it may seem somewhat obvious in retrospect, but it can be easy to get caught up with the specific implementation details).

## The solution
The key insight is that, instead of quantifying over types like many functional programming languages, we want to quantify over traits. Therefore, instead of having an endofunctor on the category of types $$\mathcal{U}$$ (as is the norm when it comes to functors in programming languages), we have a functor from the category of types $$\mathcal{U}$$ to the category of traits $$\mathcal{T}$$. (It doesn't particularly matter what the "category of traits" is, as long as you can believe that traits could plausibly form a category. We can define traits mathematically another time.) The domain of the functor will parameterise over the underlying item type (e.g. `bool` in `Iterator<Item = bool>`), whereas the codomain will parameterise over traits (e.g. the `Iterator<Item = _>`).

To define a functor trait, therefore, we need to be able to parameterise over traits. In Rust currently, it's only possible to parameterise over values (i.e. function parameters) and types (i.e. type parameters) (well, and lifetimes, but that's not relevant here). In the pseudocode below, I'm going to add "trait generic parameters", prefixed with `trait`. We also have a marker trait on traits, `Func`, which is implemented for `Fn`, `FnMut` and `FnOnce`.

Our functors, then, are going to be functors on traits, not types.

```rust
// Forget boring old traits for types: traits for traits are the hot thing now!
trait Functor<trait G: Func> {
    // This is pseudo-syntax for an associated trait.
    trait MapOb<A>;
    // I'm uncurrying `map_mor` here to avoid requiring that we add currying
    // to Rust to support this pattern.
    // Notice how the particular function variant we're using is abstracted
    // into a trait parameter on `Functor`.
    fn map_mor<A, B>(
        xa: impl MapOb<A>,
        f: impl G(A) -> B,
    ) -> impl MapOb<B>;
}

// Now we can define an implementation of a trait *for* a trait.
// This might be a slightly confusing concept at first, but the
// definitions themselves are very straightforward: we're essentially
// forwarding everything to `Iterator`, which already has all the
// information we need.
// `Iterator`'s map takes a `FnMut`, so we pass it in explicitly here.
impl Functor<trait FnMut> for trait Iterator {
    trait MapOb<A> = Iterator<Item = A>;

    fn map_mor<A, B>(
        xa: impl MapOb<A>,
        f: impl FnMut(A) -> B,
    ) -> impl MapOb<B> {
        <MapOb<A> as Iterator<Item = A>>::map(xa, f)
    }
}
```

Defining a trait for a trait is very similar to a trait for a type: the only difference is in the handling of `Self`. When defining a trait for a trait, `Self` is known to be a trait, so we can use it in bounds, or in `impl Trait`, and so on. Actually, in this first example, it's not even necessary, as none of the functions are defined for `self`, but it'll be useful in the next example.

We might more conveniently express this with the following psuedocode, assuming the API convention that if we're implementing `Functor` for `Trait`, then we can define the map on objects to take a type `A` to `Trait<A>`. Of course, in Rust, we often use associated type parameters instead (e.g. `Iterator<Item = A>` rather than `Iterator<A>`), but let's just pretend the latter is sugar for the former for now, because it makes the code even simpler.

```rust
// Here we're implicitly assuming that `Self` is a type constructor,
// by the presence of `Self<A>`. This is all pseudo-syntax anyway,
// so take it with a pinch of salt, but it works out quite nicely.
trait Functor<A, trait G: Func> {
    // `Self` here refers to the *trait* implementing `Functor<A>`,
    // as demonstrated below.
    fn map_mor<B>(
        xa: impl Self<A>,
        f: impl G(A) -> B,
    ) -> impl Self<B>;
}

impl<A> Functor<A, trait FnMut> for trait Iterator<Item = A> {
    fn map_mor<B>(
        xa: impl Iterator<Item = A>,
        f: impl FnMut(A) -> B,
    ) -> impl Iterator<Item = B> {
        xa.map(f)
    }
}
```

`Future` ([defined in `rust-lang-nursery`]((https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.3/futures/prelude/trait.Future.html))) is similarly simple to define in this way:

```rust
impl<A> Functor<A, trait FnOnce> for trait Future<Item = A> {
    fn map_mor<B>(
        xa: impl Future<Item = A>,
        f: impl FnOnce(A) -> B,
    ) -> impl Future<Item = B> {
        xa.map(f)
    }
}
```

Now if we want to define a function that's generic over functors, we can do it.

```rust
// We would never actually need to use such a function as this in reality,
// because we could just use method call syntax, but it demonstrates the
// full flexibility of this approach.
fn functorial_map<
    A, B, G: Func,
    FA: Functor<A, G>, FB: Functor<B, G>,
>(
    xa: impl FA,
    f: impl G(A) -> B,
) -> impl FB {
    <FA as Functor<A>>::map_mor(f, xa)
}
```

Using this technique, I'm assuming that if you implement a trait `S` for a trait `T`, then any types implementing `T` will also have access to all the functions (and associated types) for `S`. This would making calling the functorial map as simple as calling a method on an instance of the type.

### Monads
The technique is similarly extended to [monads](https://en.wikipedia.org/wiki/Monad_(category_theory)). Once we have functors, the rest follows quite simply. Let's see, just to make it clear. (I'll be using the more elegant syntax with `Self`-as-a-type-constructor here, but it's easily rewritten in the more explicit syntax.)

```rust
// We're parameterising over two function traits here, so that we can use
// distinct ones for the functorial map and the monadic bind.
trait Monad<A, trait G: Func, trait H: Func>: Functor<A, G> {
    fn unit(a: A) -> impl Self<A>;

    fn bind<B>(
        xa: impl Self<A>,
        f: impl (H(A) -> impl Self<B>),
    ) -> impl Self<B>;
}

impl<A> Monad<A, trait FnMut, trait FnMut> for trait Iterator<Item = A> {
    fn unit(a: A) -> impl Iterator<Item = A> {
        iter::once(a)
    }

    fn bind<B>(
        xa: impl Iterator<Item = A>,
        f: impl (H(A) -> impl Iterator<Item = B>),
    ) -> impl Iterator<Item = B> {
        xa.flat_map(f)
    }
}

impl<A> Monad<A, trait FnOnce, trait > for trait Future<Item = A> {
    fn unit(a: A) -> impl Future<Item = A> {
        future::ready(a)
    }

    fn bind<B>(
        xa: impl Future<Item = A>,
        f: impl (H(A) -> impl Future<Item = B>),
    ) -> impl Future<Item = B> {
        xa.and_then(f)
    }
}
```

And to round things off, let's define a couple of functions generic over monads.

```rust
fn monadic_unit<A, G: Func, H: Func, MA: Monad<A, G, H>>(
    a: A,
) -> impl MA {
    <MA as Monad<A, G, H>>::unit(a)
}

fn monadic_bind<
    A, B,
    G: Func, H: Func,
    MA: Monad<A, G, H>, MB: Monad<B, G, H>,
>(
    xa: impl MA,
    f: impl (H(A) -> impl MB),
) -> impl MB {
    <MA as Monad<A, G, H>>::bind(xa, f)
}
```

And so we're done. We can now define functions that parameterise over functors and use the functorial mapping and similarly extend that to monads. It requires (naturally) a little more effort than usual abstractions in Rust, because it's an abstraction over an abstraction (unlike the typical first-order functor and monad abstractions in other programming languages).

What would we require in Rust to do this (or something similar)?
- We need to be able to parameterise over traits, rather than just types, pretty much anywhere we can already paramterise over types. This means defining traits *for* traits and taking traits as generic parameters.
- For the more succinct syntax, we'd also some way to infer what sort of type constructor `Self` was. However, we can always be more explicit and avoid requiring this feature at all.

And that's it. Note that we don't need full higher-kinded types here: trait parameterising is sufficient.

## What's next?
Here I've shown how we can provide an abstraction for a `Functor` and `Monad` "higher-order trait" that works for `Iterator` and `Future`.

This technique doesn't compromise on efficiency, and it feels like the right abstraction to me, but I'm not sure how useful it is. That is, though I think we can solve this *particular* problem from the original thread, it's not an argument against the premise. That said, maybe there's some potential; it'd be interesting to hear if anyone working with very generic code has some concrete use cases. But then, we still have [quite a few more problems](https://twitter.com/withoutboats/status/1027702531361857536) to work through...
