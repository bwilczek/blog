---
layout: post
title:  "How to test Slack apps"
date:   2024-10-16 09:05:15 +0200
excerpt: A quick guide on how to mock websocket communication and test Slack applications without access to real Slack.
categories:
---

Over the years Slack has evolved from a messenger app to an entry point ,,,

Communication with Slack consists of two parts:
* Posting messages to an HTTP endpoint
* Receiving messages through a websocket

Stubbing of HTTP communication can be achieved using a plethora of libraries that exist for each major ecosystem, or `hoverfly` for multi-process environment.

Stubbing of `websocket` communication is not this popular. This post describes how to create a simple `slack_mock` service, that will act as Slack
and allow for complete coverage of E2E scenarios.

# Slack Mock server

asd

# Complete flow

* HTTP POST AUTH
