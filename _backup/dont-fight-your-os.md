---
layout:   post
title:    "Comment on 'Parallelizing the Physics Solver' BSC 2025 Talk"
tags:     programming c optimization
category: Programming
---

## Intro
![Thumbnail of the talk](/assets/img/posts/dont_fight_your_os/talk_thumbnail.png)
This blog post was inspired by watching [Dennis Gustafsson's 2025 BSC talk "Parallelizing the Physics Solver"](https://www.youtube.com/watch?v=Kvsvd67XUKw).

You don't have to watch the video to follow the content here. In short, he describes the challenges he faced in using multithreading to speed up the physics solver of the Windows PC version of his excellent game [Teardown](https://teardowngame.com/).

While he correctly identifies the performance pitfalls in his job system (also known as a task scheduler), I think his solution could benefit from a different approach to address the performance issues outlined in his talk. Just to be clear: **I'm only using this talk as an example because it was held recently**. Many games I've looked at as part of my role at Intel had similar performance problems, and I’ve written code that looked a lot like what Dennis presented.

Before we begin, note that everything here is **speculative**. Apart from what's shown in the presentation, I don't have access to the game's source code. To verify the assumptions made here, we’d ideally need an xperf trace. That said, **the profiling patterns shown match what I’ve seen in other games with similar multithreading issues**.

To keep this post focused, **I'll assume the reader knows what a job system (aka task scheduler) is**, in short: a system where you submit a bunch of isolated jobs, which are then scheduled onto worker threads that run in parallel.

### All your cores are belong to us
![All your cores are belong to us!](/assets/img/posts/dont_fight_your_os/all_your_cores.jpg)*Every gamedev ever when they start to design a new job system*

One common problem I’ve seen in many job systems is that they are too greedy when deciding how many worker threads to spawn (and yes, I’ve been guilty of this myself).

At their core, many implementations look like this:

```c
JobSystem* pJobSystem = createJobSystem();
const int workerThreadCount = GetActiveProcessorCount(ALL_PROCESSOR_GROUPS) - 1;
pJobSystem->spawnWorkerThreads(workerThreadCount);
```

Where `JobSystem::spawnWorkerThreads()` spawns **N** threads, where **N** equals the return value of [`GetActiveProcessorCount(ALL_PROCESSOR_GROUPS)`](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getactiveprocessorcount) minus one — since, as Dennis correctly states, you don’t want to hog *all* the cores. Leave at least one for the OS and other apps.

> *Note:* The open-source task scheduler [EnkiTS by Doug Binks](https://github.com/dougbinks/enkiTS), which I’ve seen used in several games, also defaults to this behavior if you use `enkiInitTaskScheduler()` or `enki::TaskScheduler::initialize()` without specifying a thread count.

This seems logical at first: it scales with core count, ideally assigns one worker per core, and minimizes cache coherence issues. Win-win, right?

Unfortunately, reality isn’t that simple. Windows and/or the hardware will most likely interfere in the name of efficiency.

### Implementation #1: Sync primitives — keep the cores awake!
The problem with spawning as many threads as physical CPU cores is that workers are often underutilized. **There’s usually not enough work to keep all of them busy.** Based on the initial implementation shown in the talk, workers go to sleep as soon as the job queue is empty. If no other thread needs that core, the CPU enters a [C-State](https://edc.intel.com/content/www/us/en/design/products/platforms/details/raptor-lake-s/13th-generation-core-processors-datasheet-volume-1-of-2/003/package-c-states/). The deeper the C-State, the longer the wake-up time — and at certain levels, the core's cache is flushed too. **You end up paying twice: once to wake the core, and again to refill the cache.**

This effect is visible in the first implementation from the talk:
![](/assets/img/posts/dont_fight_your_os/traditional_thread_pool.png)

This implementation is somewhat lock-heavy but should be fine in certain scenarios. The main thread sleeps while workers are active, which is good, and the workers idle between job bursts.

However, the profile tells a different story:
![Too many cores, too little work](/assets/img/posts/dont_fight_your_os/core_parking_issue.png)*Too many cores, too little work*

You can see gaps (bubbles) between jobs, caused by the mutex guarding the job queue. During these gaps, **the CPU core may enter a C-State**, only to wake up again shortly after - introducing latency.

Dennis correctly concludes that avoiding C-State transitions leads to a more responsive system. But there’s another point: this graph also shows that the **per-job workload is too small**. 

![](/assets/img/posts/dont_fight_your_os/scaled_first_approach.png)

Using the red `batch` bars as reference, each job seems to run for only a few microseconds. **At that point, the cost of queue access outweighs the actual job cost.** If jobs are working on isolated problems, consider batching multiple tasks per job. That way, workers stay busy longer and sleep less often.

This is indirectly acknowledged by the decision to reduce the worker count to 8, which improved frame performance:
![Less cores, better performance?!](/assets/img/posts/dont_fight_your_os/core_parking_issue_2.png)*Less cores, better performance?!*

As Dennis points out, this likely limits execution to P-Cores only. He’s right that Windows favors P-Cores when scheduling. However, **this accounts for only a tiny fraction of the performance gain**. The bigger win comes from higher per-thread utilization.

### Implementation #2: Lockless thread pool — or the wasted core
Dennis correctly identifies the issue of workers sleeping between jobs and switches to a spin-wait strategy to keep cores active until the work is done.

The revised implementation:
![](/assets/img/posts/dont_fight_your_os/lockless_thread_pool.png)

The code is standard for a lockless job queue. But the statement **"Main thread sleeps on atomic spinlock waiting for workers"** is problematic because this means **one CPU core (likely a P-Core) is just burning cycles in a busy loop**.

I assume this is something like:

```c
while (mFinishedTaskCount.load() < mTasks.getCount()) {
  _mm_pause();
}
```

> *Note:* I hope you’re using [`_mm_pause()`](https://www.intel.com/content/www/us/en/docs/cpp-compiler/developer-guide-reference/2021-8/pause-intrinsic.html) in your spin loops!

![This is fine](/assets/img/posts/dont_fight_your_os/this-is-fine.jpg)*The main thread's CPU core*

Alternatives:
- Let the main thread help execute jobs until the queue is empty, and only then wait
- Or block the main thread via something like [`WaitForMultipleObjects()`](https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjects), using events that workers signal when done

This frees up the main thread’s core for other tasks. Waking the main thread is also cheap, since other cores are already active.

---

The final implementation (where workers never sleep) is well-suited to this kind of workload, aside from the main thread still spinning. But again, you’d likely get similar performance by increasing per-job workload and letting each worker stay active longer.

There’s also an argument to be made that using *all* cores is suboptimal. Instead, a job system should adaptively limit the number of active workers based on actual demand.

But more on that in another post...
