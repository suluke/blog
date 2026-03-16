---
title: "IRrigation"
published: 2026-03-16
draft: false
description: "A short post about DRY."
tags: ['programming', 'software', 'engineering', 'dry']
---

As software developers, we often find ourselves in the situation that we want to convince our peers that something about some code we like should stay or something we dislike should go out or be done differently.
In order to do so, we need to come up with _arguments_.
Unfortunately, personal taste isn't a good argument, and someone's experience seldomly means anything to anyone but the person who made it.
So we try to come up with arguments that appear to be objective and, even more importantly, which sound _smart_.
Academia gives us the perfect toolkit to draw from here.
And so we use phrases like "Hollywood principle", "Separation of Concerns", "SOLID", "KISS" to try to intimidate the other side and feel good about ourselves.
One more element in that list is the mantra of "Don't Repeat Yourself", or just DRY in short.
And yes, trying to compartmentalize common sections of code so that they can be reused, remixed and composed well is a very important part of our jobs.
If every time we wanted to make an apple pie we had to build up the universe from scratch we'd never get anywhere.

Unfortunately, in practice I have found DRY to be overused by people, however.
I am definitely not the [only](https://news.ycombinator.com/item?id=40525064) [one](https://news.ycombinator.com/item?id=32010699) to have noticed that, so I'll not expand on this a lot here.
Maybe just a small stinger:
Most of the time where I found DRY to be cited in a situation where I didn't disagree the underlying problem was that the respective colleagues just didn't have good taste / intuition about well-composing components.

What I really wanted for this post to be about is a somewhat interesting realization I had while being WET (i.e. in the shower).
In my previous experiences I have found that when it comes to intermediate representations (IRs) of some data to be processed there isn't really a ceiling as to when it becomes _too many_ intermediate representations.
Fundamentally, this seems to be in direct violation of the DRY principle.
And yet, compiler developers - the subgroup of practitioners in our profession we revere the most - cannot get enough intermediate representations in their compilers.
Parse tree, abstract syntax tree, mid-level IR, low-level IR, target-lowered IR, assembly - these are all intermediate representations any compiled program code will go through until finally there is an executable binary produced.

It should be clear that each representation is just there to be the best data structure for a certain stage of the compilation process to operate on.
Practice has shown that splitting up the process into these stages has made the overall complexity of compilation manageable.
But each time, a decision had to be made to introduce yet another way to represent a program in memory on top of the N different ways that had already existed before.
There would have been quite the reluctance to do so as it is anything but DRY.
