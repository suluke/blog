---
title: "Willy Nilly"
published: 2026-03-15
draft: false
description: "The uninvited guest that costs my profession billions of dollars and what Hamlet has to do with it."
tags: ['programming', 'software', 'engineering', 'null']
---

Lots of things in software are binary, or at least we have a tendency to make them so.
Sure, the bit, zeros and ones, are underlying any modern computer system.
But rather than that being the root of all things to follow, I think there is yet one more level beneath it.
Too me at least, thinking in binary logic is just the easiest way to think about things in general.
Everyone is thinking in terms of black and white, good or bad, yes and no, ... all the time.
And admittedly, this has gotten us remarkably far at this point!

Please don't take this as an excuse to stick to binary-only thinking for everything, however.
I encourage everyone to challenge their convictions from time to time to see if there maybe is nuance to be had.
But let's not digress.

Binary systems can represent the the real world out there at every imaginable precision.
Not because they are binary, but because binary is enough to let us approximate anything to the level of detail we desire.
Base-2 is enough to count efficiently, thus to enumerate things.
And once we can assign numbers to (and therefore represent) anything we can think of, the rest we really care for is [decidability](https://en.wikipedia.org/wiki/Decidability_(logic)).
I.e. if a statement about $thing is true or not.
I am not a mathematician, unfortunately, so I cannot academically correctly point out the concrete relationship fot his next statement.
But I think the connection between decidability and set theory must be a fairly short one.
Because what is set theory if not deciding whether $thing is part of $set or not?
To be, or not to be, or something along those lines.

Code can be correct, or it can be incorrect, i.e. it can fall into either of two sets.
The question about which is which is, unfortunately, undecidable.
But we can use type systems within our programs to partition them efficiently into the "trustworthy" and "not so trustworthy" sets.
Note that trustworthiness is not the same as correctness and not all programs failing to typecheck are automatically incorrect.

A rather long while ago I came across the blog post ["Making sense of TypeScript using set theory"](https://thoughtspile.github.io/2023/01/23/typescript-sets/) by Vladimir Klepov.
While Typescript's type system has always been intuitive to me, explicitly pointing out the relationship to set theory in this post definitely helped me to get things click into place a bit better.
From this post we learn that, in order to construct members of the set of trusted programs in their host language, type systems make use of set theory as well by exposing programmers to set algebra primitives.
We often don't notice, though, because of how different the syntax in our programs is from... well, math really.
Also, type system designers have to be generally very careful to find the right balance between ease of use, expressivity, decidability, and generally how well their idea of "trustworthiness" overlaps with "program correctness".
Designing a type system is not a task to take lightly.
And yet, one absurd decision in type system design has endured for decades, since the mid-60s.
One, that seems incredibly hard to justify if basic logic is applied.
I am talking about a decision that its [original creator](https://en.wikipedia.org/wiki/Tony_Hoare) has acknowledged back in 2009 to have been a "billion dollar mistake".
Most readers will probably know what I am referring to.
But not everybody is following programming language design topics that closely.
And I wanted in this post to also explain (potentially as an exercise to myself) what the mistake is about from first principles so that hopefully most people could follow along.

A lot of programming tutorials would introduce newcomers to some basic operations on numbers.
They'd show you how the programming language does addition, multiplication, etc.
Then they might introduce variables as a first step to parameterize the computation at hand.
Later they show you how a function is declared, and finally you will have created something akin to the following:

```ts
function sum(a: number, b: number): number {
  return a + b;
}
```

Great stuff.
A functional function that produces some sums and surely cannot ever be misused.
Because the compiler makes sure nobody is calling it with anything but two numbers, right?
Until some day, someone wakes up and decides to commit this war crime:

```ts
sum(undefined, undefined)
```

The compiler will be absolutely fine letting this one slip through (with non-strict settings in the case of TS).
Because, you see, a lot of type systems have decided [willy nilly](https://en.wikipedia.org/wiki/Willy-nilly_(idiom)) to add an uninvited guest into the `number` type.
The `number` type does not just include all numbers.
For whatever reason there is also this weird extra guy called `undefined` in there.
Most of the popular languages in software development still do this.
If your first interaction with programming is through one of the following languages, you are confronted with a world where null/nil is just a fact of life and you may never have any reason to question it.
Something like SQL, Java, JavaScript, or even [Go](https://groups.google.com/g/golang-nuts/c/rvGTZSFU8sY?pli=1), a language having received its first stable release in 2012(!).

Btw, did anybody check what exactly happened in our code example above?
Because the fun didn't end at the misuse of our innocent little function.
The language runtime designers (who are different people from the type system designers) decided to want to go wild as well and made it so that `undefined + undefined` produces a `NaN` value.
This value, short for "not a number", is another value that the type system recognizes to be a valid `number`.
Another uninvited guest to our party.
But, in large parts contrary to our previous acquaintance `undefined`, Mr. `NaN` comes with bad manners, too!
Somebody decided that it is the reasonable thing to make `NaN === NaN` evaluate to `false`.
Also, `0 * NaN` is still `NaN`, as is `NaN / 0`.
On the other hand, `1 / 0` is defined to return `Infinity`.

What is remarkable here is that even the rust people couldn't get this right.
Not because they didn't want to.
The reason why they had to include the `NaN`-madness in their language as well for the `f32`/`f64` types is simply that every processor in the world that is remotely modern and relevant is using the IEEE754 standard for floating-point arithmetic.
And this specification unfortunately makes it so that we have all these ugly [special values](https://en.wikipedia.org/wiki/IEEE_754#Special_values) manifested into hardware for all eternity.
So, on this front, I'm afraid to say humanity has lost.

Just like it has with `NULL` in the SQL standard.
To date probably the largest-scale attempt to embed [Three-Valued Logic](https://en.wikipedia.org/wiki/Three-valued_logic) into a programming system.
And everyone hates it.
I once proposed a tabular compute language where `NULL` would be treated at least so that it is equal to itself, making use of the ugly but useful `IS NOT DISTINCT FROM` construct.
But a dear colleague pointed out to me (to great dismay on my end) that postgres cannot create indexes on that, dooming any join to be a slow nested-loop join.
Concrete sources that would prove my claims here on this are sparse.
I could only find this post on [StackOverflow](https://stackoverflow.com/questions/40830940/postgresql-poor-performance-in-left-join-is-not-distinct-from).
I also remember having read the [nabble post linked](https://postgresql.nabble.com/How-to-hint-2-coulms-IS-NOT-DISTINCT-FROM-each-other-td5928175.html) from there which since seems to have died and isn't available on archive.org either.

What are the alternatives to having special values added to all types, though?
How should we let a caller know when an operation did not produce a (meaningful) value?
The answer is [sum types](https://en.wikipedia.org/wiki/Tagged_union).
In Typescript, with [strict null checks](https://www.typescriptlang.org/tsconfig/#strictNullChecks) enabled, the following is a perfect way let callers know about the possibility of absence of a return value:
```ts
/// Will return `undefined` if `n` is not found in `ns`
findIndex(n: number, ns: readonly number[]): number | undefined { /* ... */ }
```

In rust, we have the `Option` type, which can come in the flavors `Some(value)` or `None`.
C++ gives us `std::expected`.