---
layout: post
title:  Stubbing HTTP communication in E2E test with Hoverfly
date:   2024-10-16 09:05:15 +0200
excerpt: Building a robust solution for faking HTTP communication in distributed E2E testing stack
categories:
---

Stubbing out communication with external services is a common practice in automated testing. It brings various advantages, for example:

* Reduced flakiness
* Faster execution
* Ability to test edge cases (network errors)

It is pretty easy to stub requests to remote services in unit tests, where the test runner process is executing the code under test.
There's a variety of tools that hook into the HTTP client libraries and alter their behavior in runtime, making them an important tool
in the tester's tool belt. For Ruby there VCR (more examples to come).

For E2E testing the solution to this problem is not so simple. This is because the test runner occupies a different process
then the application under test. And this makes changing the behavior of the running app (HTTP client libs) in runtime impossible.
Or at least very very hacky and standing in the way of E2E testing paradigm.

Additionally the application under test can be multi-process itself: consider clustered HTTP server and a separate service
for processing of background jobs. In order to provide consistent behavior of the whole stack all processes should be experiencing
the same responses for the same requests. Tools listed in previous paragraph won't help in this case.

# Hoverfly enters the stage

The solution to this problem is making all the processes in the stack route their HTTP communication through a proxy server
and let this proxy perform any manipulation on the responses, as required by the test scenario.

[Hoverfly](https://docs.hoverfly.io/en/latest/) is (almost) ideal tool for this purpose.
It hooks into the HTTP requests processing seamlessly, without needing to alter the applications under test.

# Settings things up

cert
`http_proxy`, `https_proxy`, `no_proxy`

# Mode: capture

asd

# Mode: simulate

asd

# Middleware

asd

# Example workflow with hoverfly-client

https://www.npmjs.com/package/@bwilczek/hoverfly-client
asd

# Why almost ideal?

As great as `hoverfly` is it comes with some caveats:

* `destination`, `no_proxy` and direct requests
* not every app respects `http_proxy`
* not every app respects default location of SSL certificates
* default certificate does not work well with time traveling
* one, huge simulation, cannot upload/delete single pairs
* JSON format holding JSON responses as a single line, often encoded. Hard to tamper with.
