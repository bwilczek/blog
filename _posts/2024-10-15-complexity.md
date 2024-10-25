---
layout: post
title:  "Practical approach to software complexity"
date:   2024-10-15 09:05:15 +0200
excerpt: A few thoughs about software complexity and why should it matter to the business people.
categories:
---

As software engineers we intuitively know what is software complexity and what impact it makes on the long term maintainability. (In case you didn’t know: the smaller the complexity the easier the code is to maintain).

In long living projects complexity tends to grow over time, and that’s normal. It can be reduced (by certain amount) by refactoring, tooling upgrade
or other kinds of redesign.
Our (engineers') problem is that such initiatives do not bring immediate business value to the stakeholders.
Therefore it could be hard to convince them that they should invest team’s effort into such projects.

In this post I’d like to provide stakeholder-friendly definition of complexity and also explain why keeping it under control is good not only for engineers’ happiness, but also for business’s sustainability.

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

This usually depends on the programming language used in the project. Different languages and different tools use different formulas,
but as long we we select one of them and stick to it we can track how the complexity changes over time.

Taking actual measurements is pretty easy and usually limited to execution of a single command. In `ruby` universe that would be [flog](https://github.com/seattlerb/flog).

Pseudocode for the calculation of an impact made on complexity by a given commit ABC could look like this:

```shell
# checkout commit prior to commit in question
git checkout ABC^

# compute complexity for that commit
complexity_before = `parse output from flog`

# checkout the commit in question
git checkout ABC

# compute complexity for it
complexity_after = `parse output from flog`

# calculate the impact
print "Commit ABC has changed overall complexity by #{complexity_after - complexity_before}"
```

To assess the impact made by the whole project simply run this formula for every commit associated with the project.

# Why we should measure it?

Because *you cannot manage what you cannot measure*. Prior to giving green light to a potential refactoring project
(with no direct business value associated) stakeholders would need a metric that could be used to assess the project impact.
They will not be willing to invest resources into vague promises that *this project will make the code better*.
However after they have been familiarized with the notion that high complexity is bad for long-term maintainability they could be more eager
on investing into a project that will e.g. *reduce the complexity by 3%*.

After the project is approved and completed it is worth to assess the long term impact it made on other, more common metrics:

* **Team velocity/capacity** - it should increase as changing less complex code is easier, giving more time to implement new features.
* **Infrastructure cost** - it should decrease as less complex code requires less resources to run on production and CI environments.
* **Engineering team satisfaction** - it should increase for two reasons:
    * engineers love making code better and letting them work on a refactoring project makes them happy
    * engineers hate working with hacky, overcomplicated code. Making the code more elegant heals that pain

# Summary

Whenever technical projects (refactoring, tooling upgrades) face resistance from the stakeholders
the metaphor of reducing complexity could convince them to reassess the priorities.
It's like reducing the *fixed costs*, or other *liabilities*: frees up the *resources* to be allocated on actual development.
It's measurable, traceable, and easy to introduce to the development process.
