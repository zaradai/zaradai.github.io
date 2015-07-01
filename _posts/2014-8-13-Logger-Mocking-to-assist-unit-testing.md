---
layout: post
title: Logger Mocking to Assist Unit Testing
category: Programming
tags: [mockito, tdd, slf4j, unit test]
---

One of the issues that really gnawed at me in the past was how to effectively unit test logging calls.  Good and effective logs are important to test and using
the TDD mantra that code is only ever written to make a failing test pass, makes it obligatory

##Testing Logs

Like many people my logger is normally a static final Logger created from a logger factory using the name of the class it is instantiated in.  With slf4j the
signature is :-

```java
private static final Logger LOGGER = LoggerFactory.getLogger(Foo.class);
```

What I wanted was a way of mocking the logger so that I can verify interactions and assert that the logging expectation was satisfied.  However to mock a
private static final is not the easiest of tasks so I would resort to injected logging and then some Guice magic to give the logger the name of class it was
being injected into.  The resulting code was testable but ugly.

One day I happened upon a blog with an inspired method of allowing me to unit test using the logger created as private static final.  Unfortunately I lost the
link but hopefully someone reading this blog may know of it and allow me to attribute the genius.

The source for the utility is within a [gist](https://gist.github.com/zaradai/646dd9d5e26631493e85).

{% gist 646dd9d5e26631493e85 %}

##Create and attach a mocked appender

The **create** method is where the mocked appender is created and wired into slf4j using its defined root logger name.  The returned appender will then be used
after the logging calls to verify interactions.

##Capture Log Messages

The **captureLogMessage** method is used to capture log events on the supplied mocked appender that was created using the create() method, return the messages
in a List and verify number of interactions on the appender, i.e. how many times was the logger called.  The default call will test for only 1 interaction
whilst the extended call allows one to specify the expected number of calls.

Example usage:-

```java
@Test
public void shouldLogErrorWhenCreatedWithInvalidName() throws Exception {
    // Create the mocked appender before you call the method under test
    Appender appender = LoggerTester.create();
    // call the code being tested
    uut.create(INVALID_NAME);
    // ensure the error was logged
    List<String> logs = LoggerTester.captureLogMessages(appender);
    assertThat(logs.get(0), containsString(CREATE_ERROR));
}
```

##In Summary

With the logger utility in place writing good log statements and ensuring they are fully tested is now an easy task to perform.
