---
layout: post
title:  "How to deal with a performance faulty app"
date:   2016-05-08 16:13:25 +0100
---

# 1. Have specific goals

If you're going to try to improve the performance of an application it's a good idea to have specific goals.

For services goals are usually in the form of latency — the time it takes for a request to complete — and load — the number of concurrent requests an application can handle.

Latency can be measured in average, percentiles, or just bottom line worst case. This is usually measured in milliseconds.

Load is typically measured in Requests Per Second (RPS) that is how many requests the service was able to fulfill on average per second.

# 2. Have a load generator

Just the same as when tackling any other kind of issue, it's useful to be able to replicate it quickly. That's why you'll want to have a script that synthesises load.

[Artillery](http://artillery.io/) can be a quick an easy solution here. What you want is a script you run that generates requests and puts your application under stress.

# 3. Measure

The next thing will measuring the application against your specified goals. Artillery (and most load generator tools) will report on latency and load by default, so if you used it as a load generator you have this taken care of.

At this point, we'll assume your application is performing below your set targets. The only possible reason for that is either, not enough load generated — in which case you need to make sure you are using a capable and properly configured load generator — or that at least one of the application resources is saturated and leading to a performance bottleneck.

# 4. Keep an eye on resources

Assuming you are in fact generating enough load, and the application isn't capable of meeting the performance goals, then the reason for that is that at least one resource is saturated.

Resources that could be in the origin of the performance bottleneck for a Node.js application are:

- CPU usage
- Memory usage
- GC activity (reflected in CPU usage)
- Event loop lag
- Network latency
- Network bandwith

CPU and Memory usage can be monitored with simple tools like `ps` or `top`. GC activity in Node can be monitored using the `--trace_gc` flag. The event loop can be monitored with something like [`toobusy-js`](http://npmjs.org/toobusy-js).

