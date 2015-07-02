---
layout: post
title: Testing Annotations
category: Programming
tags: [hamcrest, tdd, unit test]
---

Recently I had a few issues with code that I thought was fully covered with unit tests.  It turns out that the problem was with an incorrect value I had
assigned to a class annotation.  This led me on a search for a suitable Hamcrest annotation matcher to immediately fix up the lack of coverage.

After exhaustive searching (caveat: my searching prowess is limited to the first page returned by Google!) I was unable to find any suitable implementation so
the rest of this blog details a solution I wrote.

##What can be tested?


In order for the annotation to be visible to the matcher it must be available at runtime. Given unit tests are executed on running code this restriction will
not affect our ability to test.  An annotation under test must have a retention policy of [RUNTIME](http://docs.oracle.com/javase/7/docs/api/java/lang/annotation/RetentionPolicy.html#RUNTIME),
the definition is

> Annotations are to be recorded in the class file by the compiler and retained by the VM at run time, so they may be read reflectively.

In addition there is one other characteristic of an annotation that determines how we are able to retrieve it reflectively, [ELEMENT_TYPE](http://docs.oracle.com/javase/7/docs/api/java/lang/annotation/ElementType.html),
which is used by the TARGET meta-annotation type to specify where it is legal to use an annotation type. The types that are runtime specific and can be tested,
are,

* TYPE - Classes, Interfaces, enums
* CONSTRUCTOR - Constructor declaration
* FIELD - Field declarations including enum constants
* METHOD - Method declarations.
* PARAMETER - Parameters on methods and constructors.

