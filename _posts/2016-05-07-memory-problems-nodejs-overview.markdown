---
layout: post
title:  "A 10,000 Foot View Of Debugging Memory Problems In Node.js"
date:   2016-05-07 10:10:10 +0100
---

When it comes to performance of our Node.js apps, unless we are operating in a memory constrained environment (such as an IoT device or a small container), we are mostly concerned with how memory usage grows and falls over the lifetime of the process rather than how much memory is consumed at any given time.

The two patterns we want to avoid are:

1. A **memory leak**: memory usage grows steadily until Node.js cannot allocate more memory and crashes, or until the OS kills the process when it decides that it's taking too much memory.
2. **Memory pressure**: memory usage spikes under load, which activates the garbage collector. This becomes a problem when the GC kicks in too frequently - remember that while the GC is running, your code is paused - i.e. no requests can be served.

Growing memory usage can also cause the process to swap, which will degrade its performance.

## Measuring Memory Usage

To be able to tell if you have a problem, and to be able to tell if your changes to the code are working as intended, you need to be able to measure your program's memory usage.

### Measuring with ps & top

`$ ps -p $(pgrep node) -o rss,vsz`

```
   RSS      VSZ
  2376  3043880
```

This tells that the resident set size (`RSS`) is 2376 KB and the virtual set size (`VSZ`) is 3043880 KB.

`RSS` is a measurement of how much RAM the process is currently using. This
would include all stack and heap memory, as well as memory from shared
libraries if pages from those libraries are actually in memory.

`VSZ` is how much memory the process has available to it. This includes memory
that is in swap and all shared libraries. `VSZ` includes `RSS` and is always
larger.

We can use `top` to monitor our process in real-time:

`top -pid $(pgrep node)`


**OSX caveat:** OSX seems to aggressively compress memory pages of processes it deems "inactive", even when there's plenty of free RAM available. This may lead to ps/top showing a small memory footprint for an idle Node.js process, which baloons once the process is doing some work again.

### Measuring from within Node

We can also use `process.memoryUsage()` to get get a measurement from
within our Node process:

```
$ node
> process.memoryUsage()
{ rss: 13000704,
  heapTotal: 6147328,
  heapUsed: 2705800 }
```

Where `rss` is the resident set, and `heapTotal` and `heapUsed` refer to V8's memory usage.

## Seeing What The GC Is Up To

We can trace GC activity in our Node program if we run it with the `--trace-gc` flag (and `--trace-gc-verbose` if we want a lot of detail):

```bash
node --trace-gc --trace-gc-verbose
```

If we then run something like this in the Node REPL:

```javascript
setInterval(() => new Array(1000).join('hello gc!'));
```

We should see a lot of output that looks like this:

```
[17572:0x101804600]    44609 ms: Scavenge 6.7 (43.9) -> 5.7 (43.9) MB, 0.1 / 0 ms [allocation failure].
[17572:0x101804600] Memory allocator,   used:  44996 KB, available: 1454140 KB
[17572:0x101804600] New space,          used:      0 KB, available:   1007 KB, committed:   2015 KB
[17572:0x101804600] Old space,          used:   4323 KB, available:     21 KB, committed:   4894 KB
[17572:0x101804600] Code space,         used:   1271 KB, available:      1 KB, committed:   2266 KB
[17572:0x101804600] Map space,          used:    292 KB, available:      0 KB, committed:   1070 KB
[17572:0x101804600] Large object space, used:      0 KB, available: 1453099 KB, committed:      0 KB
[17572:0x101804600] All spaces,         used:   5887 KB, available: 1454129 KB, committed:  10246 KB
[17572:0x101804600] External memory reported:      8 KB
[17572:0x101804600] Total time spent in GC  : 44.4 ms
```

We can also expose the GC to our JS code with the `--expose-gc` flag to be able to force a GC cycle from our code.

## GC Refresher ##

V8 uses a heap structure to manage memory used by your application (similar to
JVM and other language runtimes).

When an new object is created in JS code, with for example:

```javascript
var x = { name: ‘Mali’, species: ‘dog’, color: ‘brown’ };
```

V8 will allocate some memory to hold the object. When this object is no longer
needed, V8 will reclaim the memory it used.

The heap is divided into several *spaces*, which include new space, old space,
large object space, code space and more.

The two spaces of most interest to us are new space and old space.

#### New space and Old space

Objects in most programs tend to be small and short lived. New space is an
optimization in organising the heap to account for that.

New space is a small area of the heap (between 1-8 MB), where new memory is
very fast to allocate. Most new objects go here.

When new space is filled up, a *scavenge* is triggered (which we saw above).
This is a quick GC cycle in the new space. Objects that survive a scavenge are
marked, and objects that survive two scavenges are promoted to old space.

After a number of promotions from new space to old space, a garbage collection
in old space is triggered. These collections are slower than scavenges as the
space that needs to be traversed is larger.

## Analysing the heap

The `heapdump` module lets us take snapshots of the objects in memory. We can use Chrome Dev Tools to visualize the dump to see the objects that were in memory at the time the snapshot was taken.

The heap viewer allows us to load two snapshots and *diff* them, only showing the objects that were allocated between the points in time that the snapshots were taken.

![Chrome DevTools heap snapshot view](https://cldup.com/oQJ_kdFVuX-1200x1200.png)

The heap viewer also allows us to load two snapshots and *diff* them, only showing the objects that were allocated between the points in time that the snapshots were taken.

![heap snapshot diff view in Chrome DevTools](https://cldup.com/YDKANMIbk4-1200x1200.png)

In this example, we can see that 813 new objects of type `smalloc` got allocated between the time the first and the secand snapshots were taken, that altogether are taking up 12.4 MB.

The two columns that are of interest to us when tracking down memory leaks are `"Retained Size"` and `"# New"` - in this instance I am choosing to focus on `smalloc` rather than `(array)` as those objects are responsible for much more of the memory growth.

### Taking snapshots with heapdump

How do we take these snapshots?

1. Load `heapdump` in your program:
  ```javascript
  // require heapdump at the top of your index.js
  require('heapdump');
  ```

2. Send a `SIGUSR2` signal to the running process to tell `heapdump` to take a snapshot:

  ```sh
  kill -SIGUSR2 $PID
  ```

  This will write a heap snapshot file to the working directory of our process (in a file with a name like `heapdump-706203888.138768.heapsnapshot`).

3. If our program is known to have a memory leak, we can run a load-test against it to induce it, watch memory usage go up, and then take another snapshot (which will force-run the GC to make sure only the objects that could not be collected are shown in the snapshot).

The difference between this snapshot and the baseline snapshot will be the objects that are hanging around and not being reclaimed by the GC.

For a complete exercise in debugging a memory leak, see [Debugging Node.js Memory Leaks - A Walkthrough](/debugging-nodejs-memory-leaks/).
