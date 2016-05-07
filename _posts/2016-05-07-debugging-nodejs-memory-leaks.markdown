---
layout: post
title:  "Debugging Node.js Memory Leaks"
date:   2016-05-07 16:13:25 +0100
---

(This is an updated version of the article originally published on the [YLD blog](http://blog.yld.io/2015/08/10/debugging-memory-leaks-in-node-js-a-walkthrough/))

Memory leaks are quite common in production applications. Fortunately, they usually aren't difficult to find. What follows is a walkthrough of an exercise that was part of the curriculum of a Node.js performance workshop that [Igor](https://twitter.com/igorsoarez), [Tom](https://twitter.com/tomgco) and I have taught as part of our Node.js performance workshops.

## The problem

We have a service running in production that implements an [RPN calculator](https://en.wikipedia.org/wiki/Reverse_Polish_notation) over WebSockets. The memory usage seems to grow constantly over the lifetime of the process, which becomes especially evident when we experience a spike in usage of the service.

The source code for the service is available on our Github under [calc-server](https://github.com/hassy/calc-server). You can probably find the leak by just looking through the code - it's very short - but the idea is that we pinpoint the leak without reading the code.

Take a quick look at [client.js](https://github.com/hassy/calc-server/blob/master/client.js) for an idea of how clients use our service.

Install and run the server locally before we proceed onto confirming the problem and analysing our Node process's heap with `heapdump` to figure out a fix.

## Confirming the diagnosis

### Measuring memory usage

In production, you would use an [APM solution](https://en.wikipedia.org/wiki/Application_performance_management) to monitor your RAM usage which would alert you to the problem.

In this exercise we will use a combination of good old `ps`/`top` and some load-testing to induce the problem so that we can confirm its presence (and to check whether changes to the code are having the intended effect).

To examine memory usage of our process with `ps`:

`ps -p $PID -o rss,vsz`

(you can find out the PID of your Node.js process with `pgrep -lfa node`)

![examining memory usage of a Node process with ps](https://cldup.com/mlQMVZ3r5z-1200x1200.png)

This shows us the resident set size (`RSS`) and the virtual set size (`VSZ`) of our process.

`RSS` is a measurement of how much RAM the process is currently using. This
would include all stack and heap memory, as well as memory from shared
libraries if pages from those libraries are actually in memory.

`VSZ` is how much memory the process has available to it. This includes memory
that is in swap and all shared libraries. `VSZ` includes `RSS` and is always
larger.

And to watch memory usage in real time, we can use `top`:

```
top -pid $PID
```

![examining memory usage of a Node process with top](https://cldup.com/nz0wCf02SY-3000x3000.png)

**OSX caveat**: OSX seems to aggressively compress memory pages of processes it deems "inactive", even when there's plenty of free RAM available. This may lead to `ps`/`top` showing a small memory footprint for an idle Node.js process, which baloons once the process is doing some work again.

(It's also possible to measure RAM usage right from inside a Node process with `process.memoryUsage()` but we won't be using that.)

#### An aside on measuring memory usage

Memory management in modern operating systems is very complicated, and there is no simple answer to "how much memory is my process using?".

What we are looking for here is to confirm that memory usage is growing under load - not precise amount of memory used.

Further reading:

- http://stackoverflow.com/questions/860878/tracking-actively-used-memory-in-linux-programs/872456
- https://mail.gnome.org/archives/gnome-list/1999-September/msg00036.html
- http://bmaurer.blogspot.co.uk/2006/03/memory-usage-with-smaps.html



### Confirming growth

The next step is to generate some load on our server to confirm memory growth. We'll be using [Artillery](https://artillery.io), a simple but powerful load-testing tool (developed by yours truly) to do that.

Our load-testing script (included in the repo in [test.json](https://github.com/hassy/calc-server/blob/master/test.json) will create 10 new user sessions on average every second for 120 seconds. Each of those users will call our service to add two numbers:

<script src="https://gist.github.com/hassy/1ad0b1bb8b92903ddfe8.js"></script>

To run this script, install Artillery:

```
npm install -g artillery
```

and then run it with:

```
artillery run load_test_for_calc_server.json
```

Have `top` monitoring your calc-server while the test is running. We should see memory usage growing steadily for two minutes.

On my machine, memory usage went from 14MB right after Node started, to 38MB. Re-running the script a couple more times took memory usage up to 110MB. Okay, Houston, we do have a problem.

#### An aside: exercise for the reader

Can we be sure that we have a leak? What if the garbage collector simply isn't running a collection because the system still has plenty of free memory?

We could find out by forcing the garbage collector to run with a couple of steps:

1. Run our server with the `--expose-gc` flag:
`node --expose-gc server.js`.
This makes the `gc()` function available to our JS code which forces a collection.
2. Create a handler for `SIGUSR2` with [process.on](https://nodejs.org/api/process.html#process_signal_events) which calls `gc()`.
3. Run the load-test again, and tell our process to run a collection with  `kill -SIGUSR2 $(pgrep -lfa node | grep server.js | awk '{print $1}')` to see if that makes a difference.

**Note for Node.js 5**: The GC seems to have changed behavior between Node 0.12 / Node 4 and Node 5 and is less aggressive about reclaiming memory now. Whereas the GC would kick in somewhere around the 70MB mark in this example app in earlier versions of Node, under Node 5 it doesn't seem to mind when the memory goes as high as 120MB.

## Heap analysis

The `heapdump` module lets us take snapshots of the objects in memory. We can then use Chrome Dev Tools to look through them to find the type of object that's leaking, which will help us pinpoint the problematic code in our application.

The steps we will take are:

1. Use `heapdump` after our program starts - this is our baseline snapshot
2. Run a load test to induce memory growth
3. Take another heap snapshot. The difference between this snapshot and the baseline snapshot will be the objects that are hanging around and not being reclaimed by the GC.

### Taking snapshots with heapdump

First, we install heapdump with `npm install heapdump` and require it in `server.js` (or `index.js` in your application).

We can then send `SIGUSR2` to our Node process which will write a heap snapshot to the working directory of our process (in a file with a name like `heapdump-706203888.138768.heapsnapshot`)

For this exercise, I am taking 3 snapshots: **(1)** right after starting the server; **(2)** halfway through the load-test; **(3)** after the load-test finishes.

#### Caveats

There are known issues with compatibility between certain versions of Node/Io.js and Chrome, where DevTools does not compute Retained Size correctly and won't show the full retainer tree. I am using Node 0.12.7 and Chrome 42 - if you run into issues you may need to upgrade Node or use [nvm](https://github.com/creationix/nvm).

Another potential issue to be aware of is that your system needs to have `2 x heap size` the amount of RAM available when a snapshot is taken, otherwise you may see empty heap snapshot files or OOM killer messages.



### Using Chrome Dev Tools

Once we have our snapshots, we can use DevTools to analyse them.

![Chrome DevTools memory profile](https://cldup.com/5bRImKHbKX-1200x1200.png)

Once loaded, we can inspect the heap.

For a refresher on what various terms like Retained Size mean refer to:

* https://developers.google.com/web/tools/profile-performance/memory-problems/memory-101?hl=en

![heap snapshot in Chrome DevTools](https://cldup.com/oQJ_kdFVuX-1200x1200.png)

Comparison view is the one we want to use to narrow down the cause of growing memory usage:

![heap snapshot difference](https://cldup.com/YDKANMIbk4-1200x1200.png)

This view tells us that compared to the fresh-start snapshot, we have 813 new objects of type `smalloc`, that altogether are taking up 12.4 MB.

The two columns that are of interest to us are "Retained Size" and "# New" - in this instance I am choosing to focus on `smalloc` as those objects are responsible for most memory growth.

![object's retaining tree](https://cldup.com/cMWwgRxNmP.thumb.jpg)

Drilling into the retainining tree for one of these objects, we can infer the presence of a function `cleanup()` that is a listener for the `SIGINT` event, that refers to a an array named `clients` which holds references to WebSocket connections. `SIGINT` (and `SIGTERM`) are the signals used to tell a process to exit by a process supervisor such as Upstart or init (or by your terminal when you press `Ctrl+C`). In this instance, the `cleanup()` function is likely to disconnect all connected WebSocket clients before the process exits. We can also guess that the server had 813 active WebSocket connections at the time the heap snapshot was taken.

Let's look at what the heap looks like after the load-tests finishes:

![heap snapshot difference after load-test](https://cldup.com/HsX2GI25QK-1200x1200.png)

Uh-oh, that does not look good. The number of references to WebSocket connections has increased by 1649, which could only make sense if there were still clients connected to the server, but as **(a)** the load-test had finished before we took the snapshot, **(b)** GC runs before a snapshot is written, we know that we are not getting rid of references to connections that close somewhere in our code (which we can now go and start looking at).


Fixing the bug is left as an exercise to the reader. :)
