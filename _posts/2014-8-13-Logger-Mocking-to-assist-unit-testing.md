---
layout: post
title: Logger Mocking to Assist Unit Testing
category: Programming
tags: [mockito, tdd, slf4j, unit test]
---

One of the issues that really gnawed at me in the past was how to effectively unit test logging calls.  Good and effective logs are important to test and using
the TDD mantra that code is only ever written to make a failing test pass, makes it obligatory