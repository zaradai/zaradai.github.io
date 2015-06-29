---
layout: post
title: Low Latency Logging with scala-logging
---

##Introduction

Low latency application developers need to think carefully about how they log; In this blogI will describe the problems and provide a possible
solution that allows loggers to be used without overly impacting the applications performance.