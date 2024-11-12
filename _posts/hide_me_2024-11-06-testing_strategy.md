---
layout: post
title:  "How to create a testing strategy"
date:   2024-11-06 09:05:15 +0200
excerpt: A list of questions one has to ask in order to create an effective testing strategy
categories:
---

This post discusses the agile approach to the creation of some typical documents that define
the Quality Assurance process in Software Development. There are multiple ways of doing this
and no way can be treated as a *standard*, or a *silver bullet*. The approach presented here sums up the experience 
gained by the author throughout his 20 years of profession career in web development.
It might not work 1:1 for you, but could provide some inspiration for improvements.

Before we start let's define some basic terms. These definitions may very between organizations.
The ones listed below may mean exactly same thing in your company, but again: they may also slightly differ.

## Glossary

### Test Strategy

An organization wide set of guidelines for creating project specific Test Plans.

### Test Plan

Describes how to test the project. Contains answers to questions like:

* What to test?
* When to tests?
* Where to test?
* How to test?
* Who should test?

Answers will differ significantly depending on the project size, scope, risks, timeline, development process etc.

### Test Scenario and Test Case

Both describe the expected behavior of a single feature. Some organizations use these terms interchangeably,
Some treat Scenario as a high-level bucket for a set of Test Cases. For example Scenario "User login" can contain a brief
description of the login feature (or links to associated requirements) and a list of associated Test Cases
for the happy path, invalid password, expired account, password reset etc. 

What matters most from the practical standpoint is that every Test Case contains the setup, actions and assertions.

Whether you call it a Test Case or a Test Scenario does not matter that much as long as the organization uses the terms consistently.

## Strategy Survey

Here's a list questions about tools, practices and other aspects that I find most impactful for the creation of a successful Test Strategy.

### Should we focus more on testing or monitoring?

In other words proactive vs reactive approach to Quality Assurance. Consider:

* How much does it cost to correct a bug that has found its way to production?
* How much does it cost to maintain an extensive test suite that will catch the bugs earlier?

For hobby projects it usually makes sense to do little formalized testing and react upon bug reported from the users (if there are any).
For life critical projects (medical systems, aerospace industry) the costs of reactive approach could be overwhelmingly high.

Modern application performance monitoring systems and automated error reporting tools provide means to detect and fix bugs fast
and are often a tempting alternative to extensive testing.

### How is the project being deployed?

Big bang release every 6 months or Continuous Delivery?

### How do we build the Test Pyramid?

asd

### Is there any UI/IX?

Web? CLI? Android? IOS? Standalone application? Visual testing?

### Does it interact with sensitive data?

asd

### Is there any third party certification involved?

asd

### Is performance critical?

asd

### How interconnected is the system with other systems?

asd

### Is there any logic driven by the passing of time?

asd

### Is there anything missing?

Your experience and organization may have some different questions to consider.
Add them to this survey and let them be addressed in your Test Strategy.

## Metastrategy

Metastrategy is a strategy for creating a successful Test Strategy.
It's quite simple: just go through the questions from the previous paragraph (Strategy Survey)
and include them in your strategy: either provide an answer that is should work
for all projects your organization is working on, or leave the question open and the the
individual Test Plan decide about the answer.

## Example Test Plan

As stated above, the Test Plan should provide answers to the following questions:

* What to test?
* When to tests?
* Where to test?
* How to test?
* Who should test?

For a typical, agile web project the structure of the Test Plan could look like this:

* Test Pyramid layers and tooling
* Scenarios:
  * automated
  * stored together with the code
  * executed on CI (PR, release)
* Monitoring solution and associated response policy
* Error reporting solution and associated response policy
 
With simplified answers being:
* What to test?
* When to tests? Continuously
* Where to test? Locally (pre-commit hooks), CI
* How to test? Automatically, different tools for different pyramid layers
* Who should test? Automation

