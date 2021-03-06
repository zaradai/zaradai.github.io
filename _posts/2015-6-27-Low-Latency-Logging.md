---
layout: post
title: Low Latency Logging with scala-logging
category: Programming
tags: [disruptor, latency, logging]
---

Low latency application developers need to think carefully about how they log; In this blog I will describe the problems and provide a possible
solution that allows loggers to be used without overly impacting the applications performance.

##Problem

One of the key tenants to low latency programming is to avoid performing any sort of IO and schedule your business logic to run entirely within memory. At the
same time however one often needs to issue log statements.  In doing so, depending on what appenders you have configured, these operations could harm your
latency.  For example, if you have an attached FileAppender, the appender will do IO and a file lock for every log call.

In most instances the configuration of logging can be changed without the original authors input and slow, latency harming appenders could be added spoiling a
well designed applications performance.

To overcome this issue a solution would be to efficiently offload the logging and associated IO calls to another thread to process.

##Solution

I decided to implement a solution using a disruptor queue to process the log requests and implemented a scala-logging trait to allow the solution to be simply
inserted wherever it is required.

With scala-logging, one can easily add logging by mixing in a logging trait.

```scala
class MyClass extends StrictLogging {
  logger.debug("This is very convenient.")
}
```

Whenever a logging request is made, the code will
- wrap the logger name, log message, log level and cause if specified into a logging event
- The event is then pushed onto the disruptor queue
- On a separate thread, the event will be read.
- Using the target logger name, a logger will be obtained (cached) and the message logged.

The code can be found in my [github](https://github.com/zaradai/fastlog) repository.

##Benchmarking

The code was benchmarked using [jmh](http://openjdk.java.net/projects/code-tools/jmh/) and I created 3 scenarios.  In each scenario, after a specified number
of operations (incrementing a counter) I would log the counter value.  Each scenario implemented the logging thus.

- Null Logging - This was used as a baseline and the log call would do nothing.
- Strict Logging - The logging performed by this scenario would mixin scala-logging StrictLogger.
- Proxied Logging - This is my implementation which hands the logging operation to another thread

Each log would create a single file appender named after the scenario and logging frequency.

What I didn’t want to do is flood the test with logging calls as this would grind the system to a halt and provide bad metrics so I used a mod operator to
spread logging calls based on the tests logging frequency.  I.e. if the rate is set to 1000, it would increment the counter 1000 times before logging.  This
way I could increase the load and see how the latency was affected. JMH has a nice feature that allowed me to specify the log granularity as a table and have
it reported.

I ran the tests using 5 pre-run iterations to warm up the code and 5 measurement runs each lasting 200 ms.

The tests ran on my laptop, an Intel i5 CPU @ 2.53 Ghz and hit an SSD drive.

##Results

The table shows for each logger how many operations it was able to perform every millisecond.

| Logger      | 1000000   | 100000    | 10000     | 1000      | 500       |
|-------------|-----------|-----------|-----------|-----------|-----------|
| Null Logger | 54514.805 | 56136.28  | 56773.805 | 55570.36  | 56077.728 |
| Strict      | 57882.823 | 48719.072 | 35887.066 | 11680.522 | 6890.911  |
| Proxy       | 58194.838 | 51543.935 | 44012.717 | 40470.767 | 30660.113 |

![Logging Results](/images/fastlog_results.png)

As you can see, the number of log operations for the strict logger begins to fall off as the load increases whilst the proxy logger is able to maintain a
decent throughput.  To prevent back-pressure on the disruptor queue I ran with 64K slots in the disruptor queue.  To see the effect of an overworked queue and
how it affects the timings I reran with a smaller queue size of 64.

| 1000000   | 100000    | 10000    | 1000     | 500      |
|-----------|-----------|----------|----------|----------|
| 58920.604 | 54193.853 | 36859.89 | 19061.35 | 9962.844 |

Clearly as the slots are exhausted pressure is placed on logging code and the performance starts to look like the strict logger.

##Summary

As designed the proxy logger is able to maintain fairly consistent latency through increasing loads whilst the StrictLogging logger shows a deterioration of
performance as the number of log events increases.

Depending on characteristics of the application, one should tweak the proxy logger queue configuration to maintain a desired latency profile.
