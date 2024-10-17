---
layout: post
title:  "Practical approach to software complexity"
date:   2024-10-15 09:05:15 +0200
excerpt: A few thoughs about software complexity and why should it matter to the business people.
categories:
---

As software engineers we intuitively know what is software complexity and what impact it makes on the long term maintainability. (In case you didn’t know: the smaller the complexity the easier the code is to maintain).

In long living projects complexity tends to grow over time, and that’s normal. It can be reduced (by certain amount) by refactoring or other kinds of redesign. Our problem is that such initiatives do not bring immediate business value to the stakeholders. Therefore it could be hard to convince them that they should invest team’s effort into such projects.

In this post I’d like to provide business-friendly definition of complexity and also explain why keeping it under control is good not only for engineers’ happiness, but also for business’ sustainability.

# What is complexity?

Let’s use the image below to explain what complexity is.

![Complexity as wiring](/blog/assets/complexity.png)

Complexity is a metric describing how difficult to change the software is.

It consists of two components:
* **essential complexity** - originates from the business rules. The very core of the system. Cannot be reduced without sacrificing the functionality.
* **accidental complexity** - originates from the implementation. Brings no business value.

The smaller accidental complexity overhead the better.

Now let’s go back to the picture above.

Think of the clean wiring as the essential complexity with very little accidental complexity added.
The system is not simple, because the domain is not simple.
It looks easy to comprehend and change though, as the non-domain overhead is minimal.

Now think of the messy wiring as the essential complexity with a lot of accidental complexity on top of it.
It does the same functionality as the first one, but is much harder to comprehend and change.

Whenever engineers say that certain initiative will reduce the complexity they mean removing the obsolete wires and simplifying the layout of the essential ones.

# Where does the accidental complexity come from?

Accidental complexity can be added to the software in two ways:
* **unintentionally**, when the engineering team does not have the experience with the
technical solutions required to complete certain task. This happens despite the best
efforts and good will displayed by the team. It can be mitigated by involving people with
more experience early in the solution design. Reaching out to architects should be a
good starting point.
* **intentionally**, when certain shortcuts are being made in order to achieve a short term
goal. Can be mitigated by being more flexible with deadlines. When in doubt refer to the
messy wiring picture and ask yourself whether adding a random, loose wire that can
make one project go live a bit earlier is beneficial for other projects in the long run.
Spoiler alert: it is not :)

# How to measure the complexity?

This usually depends on the programming language used in the project. Different languages and different tools use