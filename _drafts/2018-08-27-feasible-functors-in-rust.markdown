---
layout: post
title:  "Feasible functors in Rust"
date:   2018-08-27 00:19:05 +0100
latex:  true
---
{% if page.noindex == true %}
  <meta name="robots" content="noindex">
{% endif %}
[withoutboats](http://github.com/withoutboats), one of the Rust language design team, recently posted a thread on [the infeasibility of monads](https://twitter.com/withoutboats/status/1027702531361857536) as a useful abstraction technique in Rust, as a response to the persistence of some (usually from outside the Rust community) in claiming that "Rust is doing things incorrectly" by developing specific solutions to problems, rather than using a general category theoretic framework for everything. The points demonstrate real difficulties with attempting to use a general framework for these problems and to me serves perfectly as a "the ball's in your court now" to anyone claiming Rust is ignoring theory and coming up with unnecessary solutions to solved problems: if you think Rust could use monadic abstractions, you have to be able to address these counterarguments.

Personally, I think the design decisions behind these features are sound and that a monadic abstraction for Rust is infeasible with the design constraints. However, I think the thread is interesting, academically, as a set of design challenges. These are a set of problems, without existing solutions, that could potentially have useful consequences if tackled.

In this post, I'm going to take a look at one of the assertions and see if there's any way we might address it. In doing so, I hope to show that a functorial abstraction for existing interfaces in Rust is not as impossible as some think.

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

I'm going to make two simplifications to my treatment of this problem (both of which I think can be readily addressed, but don't have time to get into now).
- I'm going to be considering the problem specifically for a `Functor` abstraction for `Iterator`, rather than `Monad` and `Future`, because I think they're simpler. (I'll leave the extension to `Monad` and `Future` for another time, or as an exercise for the reader.)
- I'm also going to be pretending Rust has a typical function type `A -> B` from `A` to `B`, instead of directly considering the `Fn`, `FnMut` and `FnOnce` traits. As pointed out, this is making a simplification, but one I think can be addressed (later).

Okay, so that said: what's the problem? Let's get our definitions straight first.

A functor $$F: \mathscr{C} \to \mathscr{D}$$ (for categories $$\mathscr{C}$$ and $$\mathscr{D}$$) is formed from two components:
- A map on types, sending a type $$A: \mathscr{C}$$ to a type $$FA: \mathscr{D}$$.
- A map on functions, sending a function $$f: A \to B$$ to a function $$Ff: FA \to FB$$.

An example of a functor is the `List` type constructor along with `map` on lists, which:
- Takes a type `A` to the type `List(A)` of lists containing elements of type `A`.
- Takes a function `f: A -> B` and returns a function `List(f): List(A) -> List(B)`, which "maps" the function `f` over every element in its input.

Functors are often used for "mappable containers": some data structure that we can map a function over.

Intuitively, this seems to fit iterators quite well: in Rust, any type implementing `Iterator` has a `map` function, which looks just like the `map` in `List`.

However, if we take a look at the signature of `Iterator::map`, we can instantly see something's wrong.

```rust
fn map<B, F>(self, f: F) -> Map<Self, F> where F: FnMut(T) -> B;
```

Let's write this out using type notation, because I think it's clearer. I'm using $$*$$ for the category of types (or `Type` in the code examples), $$I$$ for the type constructor corresponding to the `Iterator` trait and $$M$$ for the type constructor `iter::Map`. (That traits can be represented as type constructors may or may not be obvious to you, but if it's not, I'll ask you to go along with it for the sake of this post: I'm going to explore this in more detail in an upcoming article, but it's not a digression I want to get into now.) The function `Iterator::map` then has a signature that looks something like this:

$$\text{map}: \prod_{A, B: *} \prod_{F: A \to B} I(A) \times F \to M(I(A), F)$$

Whereas for a functor, we want something more like:

$$\text{map}: \prod_{A, B: *} \prod_{F: A \to B} I(A) \times F \to I(B)$$

Notice how the return type is different: for a functor, we want the return type to be over the same family of types as the input, but for $$B$$ rather than $$A$$. However, regardless of what type `Iterator::map` is called on, it always returns an `iter::Map`: it's not as parameteric as the functorial map.

This is the fundamental problem: we want to express the type of a functor map generically, whereas with Rust it's always specialised to some specific data type (such as an external iterator like `iter::Map`). We just don't have a functor. Let's see how we can address this.

## What makes Rust special?
Why is this a problem in Rust but not other, functional languages like Haskell? It's primarily due to optimisation concerns. Rust makes performance a high priority: you should be able to safely use its high-level abstractions with little to (ideally) no cost. Naïve iterators (when chained together) are very inefficient, continually constructing and then deconstructing data structures. Rust's strategy ensures that even long chains of iterator methods are performant (this is known as internal versus external iteration).

Because of this design pattern (among others), Rust doesn't put so much of an emphasis on abstraction through types, instead preferring to use traits as the primary form of abstraction.

Thus, when considering this problem, the mistake is to get caught up in the types. There are several strategies for approaching this problem: for example, one could try to consider how we can convert from an `iter::Map` to another, arbitrary iterator, so that we can get the type we started with back (and hence restore the functorial nature of `Iterator::map` by composing it with the conversion function). This is ultimately an unilluminating rabbit-hole to go down, as it doesn't consider take the semantics of `Iterator` at all. Fundamentally, when you have an iteration, the underlying types (such as `iter::Map`, `iter::Once` or `iter::Chain`) are unimportant: it's the iteration itself that's key (it may seem somewhat obvious in retrospect, but it can be easy to get caught up with the specific implementation details).

## The solution
The key insight is that, instead of quantifying over types like many functional programming languages, we want to quantify over traits. Therefore, instead of having an endofunctor on the category of types $$*$$ (as is the norm when it comes to functors in programming languages), we have a functor from the category of types $$*$$ to the category of type constructors $$* \to *$$ (in which traits live). The domain of the functor will parameterise over the underlying item type (e.g. `bool` in `Iterator<Item = bool>`), whereas the codomain will parameterise over traits (e.g. the `Iterator`).

Our functors, then, are going to be functors on traits, not types.

```rust
// I'm going to be making use of type-level functions here, the syntax for
// which will be the same as usual Rust syntax, but explicitly parameterising
// the kinds of the generic parameters, which may be higher-order.
trait Functor {
    // This is placeholder syntax for a type-level function: `MapOb` takes a
    // type and returns a type constructor.
    fn MapOb<A: Type> -> (Type -> Type);
    // I'm uncurrying `map_mor` here to avoid requiring that we add currying
    // to Rust to support this pattern. I'm also extending the `impl Trait`
    // syntax to `impl TypeConstructor` (considering traits as type
    // constructors) to make it prettier.
    // Again, I'm also pretending `A -> B` is a proper function type in Rust.
    fn map_mor<A: Type, B: Type>(f: A -> B, xa: impl MapOb<A>) -> impl MapOb<B>;
}

// Now we have an implementation of a trait *for* a trait. This might be a
// slightly confusing concept at first, but the definitions themselves
// are very straightforward: we're essentially forwarding everything to
// `Iterator`, which already has all the information we need.
impl Functor for Iterator {
    fn MapOb<A: Type> -> (Type -> Type) {
        Iterator<Item = A>
    }

    fn map_mor<A: Type, B: Type>(f: A -> B, xa: impl MapOb<A>) -> impl MapOb<B> {
        <MapOb<A> as Iterator>::map(xa, f)
    }
}
```

We might more conveniently express this with the following, assuming reasonable API conventions (such a trait `Tr` implementing `Functor` always taking a type `A` to the type `Tr<A>`, a pattern that holds, say, in Haskell).

```rust
// Here we're assuming that `Self` is a type constructor, which means we need
// to know that we can pass a type to it, so I'm using an explicit syntax for
// a trait that may only be implemented by traits.
trait Functor<A: Type> on Type -> Type {
    // `Self` here of course refers to the *trait* implementing `Functor<A>`.
    fn map_mor<B: Type>(f: A -> B, xa: impl Self<A>) -> impl Self<B>;
}

impl<A> Functor<A> for Iterator<Item = A> {
    fn map_mor<B: Type>(f: A -> B, xa: impl Iterator<A>) -> impl Iterator<B> {
        <MapOb<A> as Iterator>::map(xa, f)
    }
}
```

Now if we want to define a function that's generic over functors, we can do it.

```rust
// We would never actually need to use such a function as this in reality,
// because we could just use method call syntax, but it demonstrates the
// full flexibility of this approach.
fn functorial_map<A, B, FA: Functor<A>, FB: Functor<B>(
    xa: impl FA, f: A -> B
) -> impl FB {
    <FA as Functor>::map_mor(f, xa)
}
```

And so we're done. We can now define functions that parameterise over functors and use the functorial mapping. It requires (naturally) a little more effort than usual abstractions in Rust, because it's an abstraction over an abstraction (unlike the typical first-order functor abstraction in other programming languages).

What would we require in Rust to do this (or something similar)?
- We need to be able to define traits for traits (which effectively means we're considering traits to be type constructors: this concept can be formalised, but that's the subject of another post). We also then need to be able to restrict what kinds a trait may be implemented for (it might be possible to define some traits that could be implemented for either types or traits in general, but it doesn't hurt to be explicit here).
- We need to be able to parameterise over type constructors (i.e. the `Type -> Type` bounds in the examples). We don't necessarily need full higher-kinded types, but something more powerful than we already have. This includes being able to treat traits as type-level functions.

And that's it. If we wanted to define `Functor` slightly more generally, we'd also need *explicit* type-level functions, but for the abstraction here, that's not necessary and we can simply make use of traits.

## What's next?
Here I've shown how we can provide an abstraction for a `Functor` "higher-order trait" that works for `Iterator`. Obviously, I haven't actually refuted the original claim: that `Monad` cannot abstract over `Future`. However, I think that extending `Functor` to `Monad` should be relatively straightforward (barring any surprises) and I imagine `Future` should work similarly to `Iterator`.

There's also the issue of dealing with Rust's function traits instead of `->`, but this seems to me not to be too difficult, though potentially syntactically noisier. In fact, if we just state that type constructors specifically *do* make use of a `->` constructor, rather than `Fn` and friends, then we only have to worry about the `f: A -> B` in the above examples, which is solved simply by parameterising over the type of `f`.

This technique doesn't compromise on efficiency, and it feels like the right abstraction to me, but I'm not sure how useful it is. That is, though I think we can solve this *particular* problem from the original thread, it's not an argument against the premise. That said, maybe there's some potential; it'd be interesting to hear if anyone working with very generic code has some concrete use cases. But then, we still have [quite a few more problems](https://twitter.com/withoutboats/status/1027702531361857536) to work through...
