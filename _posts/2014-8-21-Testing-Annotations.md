---
layout: post
title: Testing Annotations
category: Programming
tags: [hamcrest, tdd, unit test]
---

Recently I had a few issues with code that I thought was fully covered with unit tests.  It turns out that the problem was with an incorrect value I had
assigned to a class annotation.  This led me on a search for a suitable Hamcrest annotation matcher to immediately fix up the lack of coverage.