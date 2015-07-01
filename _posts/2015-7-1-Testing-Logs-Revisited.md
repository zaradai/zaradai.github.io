---
layout: post
title: Testing Logs Revisited
category: Programming
tags: [tdd, slf4j, unit test]
---

Whilst moving my old blogs to github I reviewed one I did on testing logs and I thought it good timing to present a refactored implementation and in my current
language of choice, Scala.

I have also promised myself to blog on topics other than logging and testing.

##Briefly

In a previous [post](http://zaradai.github.io/Logger-Mocking-to-assist-unit-testing/) I provided an implementation to unit test logging.  Whilst it met my
needs, it always felt a little unwieldy, mainly in the reporting of log events.  To fix this I replaced the mocking appender with a concrete appender and capture log
events in a queue.

##Breakdown

Whilst the method of intercepting logs remains the same, i.e. add an appender to root to capture logged calls, it is the method of reporting which I feel is
better handled.  So breaking it down :-

###Log events capture

Instead of creating a mock appender I create an appender that **collects** log events

```Scala
class CollectingAppender extends AppenderBase[ILoggingEvent] {
  private[this] val events = new mutable.Queue[ILoggingEvent]()

  override def append(eventObject: ILoggingEvent): Unit =
    events.enqueue(eventObject)

  def pop(): ILoggingEvent =
    events.dequeue()

  def clear(): Unit =
    events.clear()

  def size: Int =
    events.size
}
```

The collecting appender extends the abstract appender base implementation, the [AppenderBase](http://logback.qos.ch/manual/appenders.html#AppenderBase)
contains bare minimum code required.  The implementing class needs to implement an append method, in my case this method will add each log event to a queue.

I have added a number of methods to help extract captured information,

1. pop - Returns the oldest log event and removes it.  Note if no events remain will throw, which is helpful in a unit test.
2. clear - Clears out any existing event and can be used to reset a test state.
3. size - Returns number of events collected.

The size method can be particularly useful for asynchronous ops where one can use eventually to wait until the desired or expected number of logs
have been captured

###Setup code

The following code shows how one would set-up log event collection.  Normally this would be in an object

```scala
object LogCapture {
def createCollectingAppender: CollectingAppender = {
    val appender = new CollectingAppender
    val logger = LoggerFactory.getLogger(org.slf4j.Logger.ROOT_LOGGER_NAME).asInstanceOf[ch.qos.logback.classic.Logger]
    logger.addAppender(appender)
    logger.setLevel(Level.TRACE)
    appender.start()
    appender
  }
}
```

The required steps are

1. Create the collecting appender
2. Attach it to the root logger
3. Set level to finest to ensure all log events are captured.
4. Start the appender.

###Test the code

Lets say we are testing some code that emits an error log we can set-up the test like so

```scala

test("Log error test") {
  val testAppender = LogCapture.createCollectingAppender
  // execute code under tests

  // verify a log event occured
  testAppender.size should be (1)
  // verify it was wan error log
  testAppender.pop().getLevel should be (Level.ERROR)
}
```

I have created a [gist](https://gist.github.com/zaradai/66e5f5e41d23df0f0031) with this code, or you can see it being used in
my [fastlog](https://github.com/zaradai/fastlog/blob/master/logging/src/test/scala/com/zaradai/fastlog/logging/ProxiedLoggingTest.scala)