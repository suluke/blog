---
title: "We Have Visitors"
published: 2026-03-30
draft: false
description: "My view on what the visitor pattern is about and why I think it's often misunderstood."
tags: ['software', 'programming']
---

In my third semester at university it was compulsory to participate in a practical course on software engineering.
For that we were assigned into groups of five and given the whole semester to work on one of several different software projects.
My group got the task to develop an app that should teach the basics of [lambda calculus](https://en.wikipedia.org/wiki/Lambda_calculus) to children by portraying abstractions as alligators and variables as their eggs.
This idea was based on a [post](https://worrydream.com/AlligatorEggs/) by [Bret Victor](https://en.wikipedia.org/wiki/Bret_Victor).
While it was a bit quirky, the app we created came out quite nicely and satisfied our advisors to the degree that we were asked to [present it](https://www.youtube.com/watch?v=pb1G23i5mBU) during an open house event of our university.
It's still possible to find all the code we wrote during and after the project on [github](https://teamcroggle.github.io/).

As part of the implementation process, we had regular check-ins with our advisors where they would give us some guidance on how to structure the coding process and the code itself.
One of the lessons from that time included the tip to use the [visitor pattern](https://en.wikipedia.org/wiki/Visitor_pattern).
That was where first I learned to appreciate this pattern and to this day I think it addresses a very important issue in software engineering in a clever and arguably elegant way.
However, I encountered some ... _disagreements_ on what this pattern is and isn't in my professional career since then which has somewhat dampened my enthusiasm for it.
I have also found that it can be "dumbed down" a bit in practice to make it more obvious what's going on.
But before I elaborate on this, let's first take a step back and quickly reiterate the problem the pattern is intended to solve.

:::note
This is of course _my_ understanding of the pattern.
I think in all instances where people were of different opinion they were arguing that there was _more_ to it than what I believe it should boil down to.
I will address these "omissions" later in my post and why I think it does not make sense to entangle them with the core of the visitor pattern.
:::

Wikipedia says the following about the visitor pattern:

>A visitor pattern is a software design pattern that separates the algorithm from the object structure. Because of this separation, new operations can be added to existing object structures without modifying the structures.

I think, speaking abstractly, this is actually a very good and concise description of the pattern.
However, maybe we can be a bit more concrete by going through a practical example.
Let's assume we are modelling an _animal shelter_.
There are a couple of things we need to be able to do with the animals that are brought to our attention.
For example:

- `house` animals depending on their needs, i.e. putting them in an aquarium if they are fish etc.
- `feed` them
- `healthCheck` them
- give them `specialCare` in regular intervals
- ...

This list will grow for sure once we actually get into business.
But we have to start somewhere.
The intuitive first solution might be to make all these operations available as _methods_ on the `Animal` interface that all animals have to inherit from.
Unfortunately, this causes a couple of problems.
First off, we don't _own_ all the implementations of animals in the world - nature does.
So we can't just enforce all animals to implement our custom interface.
Secondly, it will cause quite some noise in the animals' implementations to implement all our operations and a lot of churn every time we need to modify or extend the operations we need to have available to run our animal shelter.
What we need, and Wikipedia already said it, is to "\[separate\] the algorithm from the object structure".

But next we run into a different problem:
We of course want to continuously make sure that every operation we are regularly going to need is supporting all the different animals that our shelter supports.
Ideally, once a new animal is brought to us that we haven't seen before _something_ should make sure we cannot forget adding support for it in all the necessary places.
As for all things that we want to be ensured in our code, that _something_ that checks it should be the compiler.
So we need a way to let our operation coding signal to the compiler which animals it supports so that it can figure out when there is something missing.
Now _this_ is the place where - in the classic visitor pattern - an interface comes into play.
For each animal variant there is going to be a `visit` function in the interface that communicates "I, the operation, know how to visit (i.e how to _deal with_) _this_ particular animal".

In code, this could look somewhat like this:
```ts
class Cat {}
class Dog {}
class Fish {}

interface Visitor {
  visitCat(cat: Cat);
  visitDog(dog: Dog);
  visitFish(fish: Fish);
}

class HouseAnimal implements Visitor {
  visitCat(cat: Cat) { return 'cardboard box' }
  visitDog(dog: Dog) { return 'kennel' }
  visitFish(fish: Fish) { return 'aquarium' }
}
```

Now there is just one final piece missing.
Namely, how to bring arbitrary animal instances together with their respective method implementation in the visitor-based operations.
The solution to this is called ["double dispatch"](https://en.wikipedia.org/wiki/Double_dispatch).
Quoting from Wikipedia:

> double dispatch is [...] a mechanism that dispatches a function call to different concrete functions depending on the runtime types of two objects involved in the call

In our case, these two objects are 1. the animal instance and 2. the operation instance.
The second one we of course get for free in most languages - calling the `visitCat` function of a `HouseAnimal` instance is trivial as long as we know the operation we want to use is indeed `HouseAnimal`.
And even if we didn't know which `Visitor` implementation we have at hand, most modern programming languages conveniently hide the dynamic dispatch in the background for use.
Dispatching based on the animal instance is the harder part.
Back in university, we were taught that using `instanceof` is super bad.
Honestly, if you keep your inheritance hierarchies flat (as you should) I don't see a big issue with it, though.
So our first option would simply be something like this:
```ts
type Animal = Cat | Dog | Fish
function dispatch(animal: Animal, visitor: Visitor) {
  if (animal instanceof Cat) {
    return visitor.visitCat(animal)
  } else if (animal instanceof Dog) {
    return visitor.visitDog(animal)
  } else if (animal instanceof Fish) {
    return visitor.visitFish(animal)
  } else {
    throw new Error(`Unhandled animal: ${JSON.stringify(animal satisfies never)}`);
  }
}
```
IMO, this is perfectly fine as long as we have the `satisfies never` guard in place there, making sure we handle all kinds of incoming `Animal`s.

If we allow ourselves a bit more extra information on the "object structure", we can also go e.g. with type tagging:

```ts
class Cat { readonly type = 'cat' as const }
class Dog { readonly type = 'dog' as const }
class Fish { readonly type = 'fish' as const }
type Animal = Cat | Dog | Fish
function dispatch(animal: Animal, visitor: Visitor) {
  const { type } = animal
  switch (type) {
    case 'cat': return visitor.visitCat(animal);
    case 'dog': return visitor.visitDog(animal);
    case 'fish': return visitor.visitFish(animal);
    default: {
      throw new Error(`Unhandled animal: ${type satisfies never}`);
    }
  }
}
```
[TS Playground](https://www.typescriptlang.org/play/?#code/MYGwhgzhAEDCYBdoG9oCcCmYAmB7AdiAJ7QJEAOG0AvNAOTCJ3STTAERIC+AUKJDAAiuAOYp0WPIRJlKNenhHNW7fJ2i9+UaADEAlhAAW4zDgLFSFKrToAzA4eUxV63rKoBBfHoC2YEPLwSAA+0MJiofpGPLYArvjACHoE0NgG5IjAhgAUYN5+IABc0F6+-gA00ABuBnoIuGjFAGq19WgAlCg80GwcSKjuGvJ5ZSDd0BAA7nVZ0NnuncjjPYwQVAxMxZgIsWj41a0NAHQ1EHVBufn+7QDcy2yQ64p0Wxg7ewdnbSet4Zejt3uq3W9iMLwk732pzqx2hCCiORGBUBPR62AwtjAsRACGKS1RqIQhjQuEm0HwGDJAFE0CS0NkAAYAVXwhjy2BAGGwLCuRWgABJkIMIIgDPYMDAKVUMGguAyUajeD1eFwgA)

If we have even more influence on the definition of our object structure, we might even want to do what we got taught in university:

```ts
class Cat {
  accept(visitor: Visitor) {
    return visitor.visitCat(this)
  }
}
class Dog {
  accept(visitor: Visitor) {
    return visitor.visitDog(this)
  }
}
class Fish {
  accept(visitor: Visitor) {
    return visitor.visitFish(this)
  }
}
type Animal = Cat | Dog | Fish
function dispatch(animal: Animal, visitor: Visitor) {
  return animal.accept(visitor)
}
```
[TS Playground](https://www.typescriptlang.org/play/?#code/MYGwhgzhAEDCYBdoG9oChrTMYBTADggBQBuAlhGQgPYBOAXNAGoVV0CUK6mmtuCAV1oA7aOUo1aAOnFV4xBAAsKnDNAC+aTaEgwAItQDmXNdjyFSrSYxYSOJntD6CRYq3RlWDhokpXcNLTQdKGgAMQpFBywcAmJZa2Z3Wk5UNV5+IVEEjwSIiEVfZQhVTE1NBABPfFxoAEFhMgBbMBBoAF44RGgAH2hvXvDItAAzAWFgBDJqUQATCnxEYEKwRpaQRgbm1oAaNzsGJIPUtWcsrDXWqTM4y2OtIA)

However, I don't like this too much for a couple of reasons:

1. There is a relatively meaningless extra function on the public interface of every animal
2. The object structure now needs to know about the `Visitor` interface
3. The dynamic dispatches in both `animal.accept` and `visitor.visitXYZ` do quite some heavy lifting.
   This can be confusing to readers. Furthermore, they aren't "free" if you care deeply about performance
4. Dynamic dispatch does not work well with generics.
   While not shown in the code examples above and not really an issue in Typescript, generally it is desirable to be able to choose the return type of an operation freely.
   In languages where generic code gets _instantiated_ on each type parameter assignment it is generally not possible to also have it be the target of a dynamic dispatch.

Especially point number 4 makes it really hard to recommend this approach.
Personally, I'd rather go with the type tagging approach instead.
Check out this code on [TS Playground](https://www.typescriptlang.org/play/?#code/MYGwhgzhAEDCYBdoG9oCcCmYAmB7AdiAJ7QJEAOG0AvNAOTCJ3STTAERIC+AUKJDAAiuAOYp0WPIRJlKNenhHNW7fJ2i9+UaADEAlhAAW4zDgLFSFKrToAzA4eUxV63j1lUAgvj0BbMCDy8EgAPtDCYmH6Rjx6+AgYaLZgwFQAagZ6CLhoADwASgB8KDzQ0ABumQjBABSMCABccIgAlE35ANylFVURNYpNEW3Qnd2VEFnRhjX2Rk1Tw6NutgCu+MAIegTQ2AbkiMCGBYU1YD7+IE3efgEAND0T2WhNGY85xy0lZS5IqB4a8jONxA3QgAHcsodoDUPJ9kN1vpAqAwmE1MAgVmh8A8sjkAHTjLK1IEXFpdMqIiDIxR0NEYDFYnFPAm9USnc4BMkIthI+izRx0hnYwnMkVTdnArkUnYYZIrECNL7S0iGNC4MHQfAYDUAUTQarQNQABgBVfCGM7YEAYbAsDmXaAAEmQ-wgiAM9gwMC15USXCNUopvDKvDcWhgAAlcCsqdcLtA-ORrb4MPEYK9cXlOGg4iJivCyiLavUmsFhtncyZ6ZjsSi0NgAEa4MD16BNgAezGDTL6A3ConLCBz+DEqHRNfoAGtU1qQF2xlVxfz5g5B8PRxIhfQwABHFYtvQrXzzsPgbQ6DA2uMBBO+JMYFNp6AZp65cq4PTYfMLx7FxCl1omnfT8UG7EVe1EQYByAj9bWQMDFwcGYHBXIxhmAuDQx4PgOFwa08RANldggfYEEOGotQ1PoWnuSjoCjGMMGvEAahaNiOiAA) for a full-fledged example.

So this was a quick run-down of what _I_ believe the visitor pattern is largely about.
Where might others have different opinions on this?

The first argument I could think of here is how much _centralized knowledge_ there needs to be regarding the shape of the object structure.
I have seen people argue that it should be possible to completely do without listing all the animals in a single place.
I.e. in my code examples, the following should ideally disappear:
```ts
type Animal = Cat | Dog | Fish
```
Also, maybe the visitor interface could be rewritten to more simply state that it can deal with all kinds of animals.
So something like this:
```ts
// This is now a base class
class Animal {}

interface Visitor {
  visit(animal: Animal)
}
```
I will admit, if you move the goal posts far enough, something like this can work for varying definitions of "work".
Of course, initially it seems a bit absurd to assume some operation can work generically over any kind of animal it has zero knowledge of.
And that is kind of the contract we have been given here, right?
All that an implementation of `Visitor` may assume on its argument is that it subclasses the `Animal` base class.
But what if we never wanted to handle all possible incoming variants of animal but just the ones _we_ explicitly know and care about?
Of course that is then a totally different problem from the one we started with (being an animal shelter for all animals brought to us, accommodating our code if needed for any newcomers).
Then basically what we want is to null-route or reject all the things we don't want/care.
That is fine - but I wouldn't call it a use case for the visitor pattern any more.
Luckily, I've only come across this opinion once in my career.

The second understanding of the visitor pattern which diverges from the one I laid out above has been much more prevalent, however.
I've seen this three or four times across different code bases now minimum.
I attribute it coming from the general setting the visitor pattern is usually employed in.
And that setting is _graph or tree data structures_.
In the app we programmed, our domain object structure with alligators and eggs really was a model of a syntax _tree_ of lambda calculus.
And whatever operations I wanted to perform, I usually didn't just want it to be performed on individual nodes in that tree but rather the whole syntax tree.
So in addition to acting on node types, I needed to traverse the tree and keep track of inter-node relationships.
And there is where I think a mistake is easy to happen:
Conflating tree traversal with node-type-based dispatch (the thing the visitor pattern solves).
[Wikipedia](https://en.wikipedia.org/wiki/Tree_traversal) list 3 * 2 = six different ways to traverse (binary!) trees.
To me, that is a big hint to try to encapsulate the complexity inherent to this somewhere separately.
In other words, if your visitor ends up having dedicated `visitBeforeChildren`, `visitBetweenChildren`, and `visitAfterChildren` callbacks, I think you are doing it wrong.
I do understand that sometimes it needs to be possible to inspect the current traversal state in order to make the decision.
And if this works for you who am I to tell you to do it differently?
I just find that _knowing_ that you have _tree traversal_ and _type-based dispatch_ as two distinct problems on your hand may lead to code that is structured more clearly and thus better to reason about.

FWIW, I coded up a slightly more complex application of the visitor pattern including a tree and state-tracking depth-first traversal in [this playground example](https://www.typescriptlang.org/play/?target=9#code/PTAEFpM0HkCMBWBTAxgF1AZzQJwK7p45IRTgBQaAngA4kDCAhhgLygDeo1dAXKAOQpm-UAF9KtEgBEA9gHNQbTtyR9+AE3kjxK0ADEAlpgAWijl0lqAZkePaJdUAEEAdgYC2jADZmmGAD6gsgqBhibkDiROAI54jDgGeO5mypYCjLHxie78ANygNiZ8YcYA2gC6YpGgTDjqcDLx6gBCMgAeKRa8AkJ1DU3gDW15oEJofH5VugDSSC4uSD5KXaoCANZzC14jmnKYfMEVU5KgABKMcAZozGYxcQlJoIG19Y11rR2Bs-OLEboASjIZMllio1DggTl8sYLldmPszrDrmgjjoTgBlYyLNBIHCdMECEzY3EjCHAhGA4Go6quDzeAAyMjGBhkLjMmOJeMClOSgXOl2RT2cbk8XgiIFIUFAADUjFcZHiaMwcTg2ZgkGg8DRJZByAYXCqrIwUCRZZh5TgADz-AB8HHIoFAADc5Wg-AAKMYTZgASj4-wdztdwXduwO8j9oADjpd5rQJXdhWMxVskejQbjdyySXdGXu2T4WYe7jTgdjVxe-Xe7XdQ29fTeLXapZjru+W3dGx+Xj47cWLYzVx57rJ7n9kIH5bQHK8KvdRNnuL4M5VpfE5CseBc6BZbPURiVaBQxmtNtzPeFdK8jOZrIANIO0Aq+GaLaeffbHShWdhzLpRGY3iBpgADuVzHqA7oqB+7CBl+jDqj0wh8MQmqqo+CoAHRTh63g+nBowISQGhaChGpEGyU5YVOIZ4QRQiIfwSb8GRaGUa61GugmdGOvBjF5tmOSsRRGE4NhrpFtk574bxhGMb0rwDEMLGgKhIlUWJOFNFWTZtNJ9FEesmyLCpanoRp4lxn2Xj6bJDHEaOpnkeZHGaa6w48bx9mElii44E5bGiZZVwrritm8eoSBGngs58LBsmOmgxgQiBoALKlACiOAQjg7oAAYAKouDCLjqF4SDqKAjAit4fAACTsLomDMEYNhIJgaVIE6uKiHlMm8eIjriOuEpkOAzheHICpXMYY5BEgNBJeAhg4L+6JIPEEFgUloAbTgXhUOASBtFcoAeDQ5XuHMOKVQhok6hQuhSFYmDosiJDLIeyaXqKN4tay1JPS96IGHILjeJ0FwKmgAD8fANDI5XVccjjPZgTBeF4cDGmsZjzu9BzA+9H4sHaaMg2DQHfi4v4wNMKTiNTv4uEC2psO6JN2nT5AoF4CEdVIejomd7gXUgV0Gh1r5Pla5Og+DXh2vFoA0AkTrMCQxCMJoLgHVgBNBETGudF9fDUoGqsGOrOKjD+uAEDL7qW9bmsbTresoHAhPo94WM4zB66OtgLUoFwOCMN1q1IOehY1deTL-S4D6e97GN+ygawwQR+6YIex7ng+6VBELnpwD6-WByravG9txgxz9DIJ2gu4PpgeBwBCeDNwsfAc4oZPA-LeHexTCufrxSVGJhwc4phX1z3gJjhV+duqe1ZiT5gmGe1BxhTzPSD9Y6BhWJBxBb1DOBoFnCVcHvW8H3PzDGHPMg0BzBGOmZbLnwRg1eavNuHd15sCAZ3bu0cfS5AIpvae70n5JVfu-I+a9ApgPaphS+sMsDt3PqAMimBAyVy0mgT0zBvTX3Hl-ZybJYG1zIWgQurMK5lmDPIUM8hwxyBvrxb+d8p70N2Ewt+LDWxxgTEmFMJgeHUMCnQmaiZbDCJoKIx8kkcwCWLIWTIxZIxy0pj4ZWsiRLyKSrmHR2QHx91JlQ3iVgFSQSZhgJMoAZCn00dkTCSYZEJScWvDqbAc55zrkmB8m8UGyRPmfDBWCfG3z4b-W+Q1P5VHiTQ2A0w-6qK0g2JoHxaztHrIpasbQ9GDwMbYvhpi65DCsZzUAQTmD5yGNvZgYT77lyIawqyxkbJdi2L2XpZTXpD0MQRKp99ML0P6YsOp-dbGOnsXiT0q9diuNPjMrwmFdiYDiQAmmGA8GBIPE0uuQj+G7JScfU+I4YkNCvns2SCT2pXNSQlf+Tz0nc1kqIbJ7lIQjkhOOYEwzR4QyMagkxkz6GjjmTYiFiyHErIOaAGEAobhuNUpCTCaK4RoF2Qs-Zv4jkNJOUeOuuLkTtKMBE3iUTbkX3uZQhFCVnmEKSW8n5KS+HfIGn8uMoVcoLhVMuXyq4R6jMqek6p84xW4jhUrFJSzHGr1HOsrAcqxKjgJSyleKKSWNPJYC4E1LLkcvpefTBTLHm8PSYkpJHyBrcq+Zkn5LCRpgDGqAAqzU5AkExYLYWtcw4R1xM1HwB8zo0zoDuVkD0NxbljWyGwpUAAKz9ZWclFZyB8tUG7x1vC4SMtJfpN13EcQIW5IopoquPcqGBTb5r+s3AGlRK2lSivqWtbAq2doWOoQMgbMK4FDVHTNfkrHsEbTQX58zOAEXpTQTBpDwAAEYfQAEJFAsDYHhcwKSvpmFKJhE9NByj5GdYFTgWC+D2xIABP+u0vCIRZXw9gjr-6-MDHwr65APUPVAAAFXahgJKJBvyRXjTzQBmrs1+RSIGAk-BhUkjvN+yECJSgERZUhxyaHb6UvhGbV5uqJ5pH4B4pI-B8McoKLYYjtHeKkdvkh5iryuWMfKK80QNHb7MZWGoBSOlBjtGo+xwi4xCUsfI2Mfg4nHUcYSlxxTXH1x+LGHjFDWqMOlAAAzlBxUieEpRV2VDupWRsHwfStLQNBlFYx007TYCm9Qjm65aeTr6OzmBEZIEwl4dhJ6bNuY3ZhTw78IY2O8MOyQ5cgA).
I hope you could find this helpful.

Cheers!