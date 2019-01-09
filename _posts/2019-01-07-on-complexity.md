---
layout:      post
title:       "On complexity"
date:        2019-01-08 08:00:00
summary:     "Thoughts on complexity of systems, separating essential and accidental complexity and why simplicity is not easy."
categories:  complexity simplicity principles
---

> "The biggest problem in the development and maintenance of large-scale software systems is complexity — large systems are hard to understand." — Ben Moseley and Peter Marks

### The ceiling of complexity

In the preface of the *Domain-Driven Design* book, Eric Evans mentions *the ceiling of complexity* in passing.

We can think about it as a natural limit of how complex a system can become before becoming intellectually unmanageable.
As we approach this limit, the system is more and more difficult to develop and maintain, causing us to expend huge resources or gamble on our changes not breaking anything.

### Essence and accident

A useful mental model for thinking about complexity is dividing it into two kinds: essential and accidental.

**Essential complexity** is inherent in the problem being solved.
In other words, it is *the heart of software*, a reflection of user expectations and intricacies of the real world, that we would still have to deal with even if we had complete knowledge, perfect tools and infrastructure.

**Accidental complexity** is then all other complexity.

Expressing some idea in a programming language adds some accidental complexity. So does using a library or framework. So does every line of code that can be simplified or removed, every workaround, every purely technical feature.

### Complexity breeds complexity

What is worse, complexity breeds complexity.

Not being able to clearly understand the system makes us duplicate code, come up with poor designs and abstractions, or place them in suboptimal locations.
Using outdated technology adds otherwise unnecessary constraints, limiting our options and forcing to use workarounds.

All of the above introduce further accidental complexity, compounding our difficulties.

### The goal

It seems natural that **our goal should be to eliminate as much accidental complexity as possible**.
This way we can both make the system easier to understand and make more room for the essential.

However, why is then dealing with accidental rather than essential so much of what we do?

### Simplicity is not easy

> “I have made this letter longer than usual, only because I have not had the time to make it shorter.” — Blaise Pascal

The first solution is rarely the most simple, particularly if there is existing complexity, time pressure or we are new to the problem domain.

There is one important distinction - *simple* is not the same as *easy*.
- **Simple** is the opposite of **complex**. Both of these words express **complexity**, which is objective and describes to what extent the parts (e.g. concepts and responsibilities) are intertwined or connected.
- **Easy** is the opposite of **difficult**. Both of these words express **effort**, which is subjective, meaning something difficult for me might be easy for you.

Finding *simple* solutions and concise ways to express them might be difficult.
It might require significant effort to reflect and iterate on the solution in order to untangle the complexity, identify and separate the concerns and express them clearly using fitting technology.

On the other hand, an *easy* solution that just works, might carry a lot of unnecessary complexity with it.
We rarely consider that a familiar, available, succinctly described solution might make us end up worse than before.

### So where does this leave us?

There is no *easy* way out of this, no silver bullet.

But we can be mindful of accidental complexity in our systems.

Once we learn to spot it, we can rally our peers to practice simplicity with us — both by trying to come up with better designs and writing cleaner, more expressive code.

Improving our collective skills makes simplicity easier to achieve, ultimately making it easier for us to cope with our systems.

### Exploring this further

1. [No silver bullet: Essence and accidents of software engineering](http://worrydream.com/refs/Brooks-NoSilverBullet.pdf) - essay by Fred Brooks, 1986.
2. [Out of the tar pit](http://moss.cs.iit.edu/cs100/papers/out-of-the-tar-pit.pdf) - a paper by Ben Moseley and Peter Marks, 2006.
3. [Simple made easy](https://www.infoq.com/presentations/Simple-Made-Easy) - a talk by Rich Hickey, 2011.
4. [Domain-driven design: Tackling complexity in the heart of software](https://www.goodreads.com/book/show/179133.Domain_Driven_Design) - a book by Eric Evans, 2003.