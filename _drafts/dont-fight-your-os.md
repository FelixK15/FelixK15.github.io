---
layout:   post
title:    "Don't fight your OS - how to optimize multithreading workload"
tags:     programming c optimization
category: Programming
---

## Intro
This blog post was insipred by watching [Dennis Gustafsson's 2025 BSC Talk "Parallelizing the physics solver"](https://www.youtube.com/watch?v=Kvsvd67XUKw). You don't have to watch the video to follow the content here - In short, he describes the challenges he faced with utilizing multithreading to speed up the physics solver of the Windows PC version of his wonderful game [Teardown](https://teardowngame.com/). While he does correctly identify the performance pitfalls with his job system (also known as task scheduler) implementation, I feel like his solution could benefit from a different approach of how to overcome the performance problems outlined in his talk. And just to be perfectly clear, I'm only taking this talk as an example because of how recent it was held. Many games I've looked at as part of my role at Intel had the performance problems outlined in Dennis' talk and I wrote code myself that looked very similar to what Dennis presented.

So I want to use this blogpost to outline the typical pitfalls with common job system implementations and how to get better performance out of your job system.

> *Note:* This post will focus on building a job system for a PC game running under Windows. I'm too inexperienced with Linux to comment on how Linux works internally and I'm pretty sure everything discussed in this blogpost isn't relevant to game console development in general.

**ADD NOTE SAYING THAT THIS IS ALL SPECULATION AND TO BE SURE WE NEED ETL TRACES - BASICALLY THERE'S LOTS OF ASSUMING GOING ON HERE BUT BASED ON DATA I'VE SEEN IN OTHER GAMES WITH SIMILAR PROBLEMS**

### CPU Cores
Before we start to talk about the software side-of-things, we have to talk about the hardware. More precisely, about the CPU that was used as the basis for the performance metrics mentioned in the talk and why that matters.

The performance measurements that are shown during that presentation (and which I'll reference in this blog post) were taken on an **i9-14900K Intel CPU**, which is a hybrid CPU. In practise, this means that the CPU has 2 different cores that it's using to execute instructions. There are **P**erformance cores and **E**fficience cores (commonly referred to as P-& E-Cores), with the general, overdramatic, consensus being that P-Cores are fast and favorable and E-Cores are slow and lame. While the E-Cores *are* slower, it's not "half of the performance of the P-Cores", as was stated during the talk.

In fact, looking at [the specs for that particular CPU](https://www.intel.com/content/www/us/en/products/sku/236773/intel-core-i9-processor-14900k-36m-cache-up-to-6-00-ghz/specifications.html), in terms of raw clock speed, the E-Cores are around ~75% as fast as the P-Cores.

For completeness, we also have to consider the cache-setup of P-/ and E-Cores since that will help us later on to understand how threads are scheduled on hybrid-CPUs.
![i9-14900k cache setup](/assets/img/posts/dont_fight_your_os/hybrid_cache_setup.png)

From this poorly structured image we can see that, for an i9-14900K, **each P-Core has 80KB of L1**, which is little bit less than the **E-Cores 96KB of L1**.
**An E-Core cluster, consisting of 4 E-Cores, shares one 4MB L2 Cache** where as **each P-Core has it's own 2MB L2 Cache**. Finally, **all cores share a 36MB L3 cache**.

The Windows scheduler assigns threads to run on CPU cores. Ideally, we want threads to run on P-Cores, if they are available.
On Intel CPUs, the following priority system decides which CPU core to run a thread on:

1. Run on P-Core, if not available
2. Run on a single E-Core within an E-Core cluster (to minimize the contention on the shared L2 cache of the cluster), if not available
3. Run on an E-Core within the lowest utilized E-Core cluster, if not available
4. Run on P-Core SMT.

**Foot-note about intel thread director**

Now that we've learned about the hardware, we can talk about the software.

### What *is* a job system?
Before we start to talk about the pitfalls, it's important to classify my world-view about what a job system is.

Basically, it's a way to distribute self-contained *jobs* to several *workers* - each worker is typically run within a thread so that all workers can run in parallel on a given set of jobs.

If you consider this graph:
![A 1000-feet view of a typical job system](/assets/img/posts/dont_fight_your_os/job_system.png)

This is a 1000-feet view of a typical graph of modern games. You have a main thread, that start to do some sequential work (eg: read player input, send network packets etc), followed by a phase of parallel work that is being worked on by all available workers (eg: Entity-Component updates) and then followed by another sequential work phase (eg: send graphics commands to GPU).

Typically, you want all workers to be finished before you start the sequential phase followed by the parallel phase where all workers are utilized. So it's necessary to serve jobs to the workers and to have a mechanism that tells the main thread when all workers are finished.

### All your cores are belong to us
![Cats is way too greedy](/assets/img/posts/dont_fight_your_os/all_your_cores.jpg)

One of the more basic problems I've seen with many job systems is that the system is too greedy when it comes to set the upper bound of how many workers to spawn (And I'm 100% guilty of this myself in one of the job systems I wrote in the past for the reasons listed below).

At it's core, many implementations look like this:

```c
JobSystem* pJobSystem = createJobSystem();
const int workerThreadCount = GetActiveProcessorCount(ALL_PROCESSOR_GROUPS) - 1;
pJobSystem->spawnWorkerThreads(workerThreadCount);
```

where `JobSystem::spawnWorkerThreads()` will spawn *N* threads (for example using [`CreateThread()`](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread)) where N is equal to the return value of [`GetActiveProcessorCount(ALL_PROCESSOR_GROUPS)`](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getactiveprocessorcount) minus one, since, as stated in the talk, you don't want to hog *all* the cores and leave at least 1 core for the OS and other applications.

> *Note:* The open source task scheduler [EnkiTS from Doug Binks](https://github.com/dougbinks/enkiTS), which I've seen being used in some games, is also guilty of this if you use the `enkiInitTaskScheduler()` or `enki::TaskScheduler::initialize()` API w/o specifying a thread count yourself.

Logically, this makes perfect sense at first. This nicely scales your job system with the number of available cores, with the idea being that each worker will run on it's own core, thus maximizing thread utilization, which should result in the best perf possible - Commit-and-done!

![](/assets/img/posts/dont_fight_your_os/clown_all_cores.jpg)

Unfortunately the reality isn't that simple and the OS will most likey get in the way of your ideal world in the name of efficiency!

### Cores want to be kept busy
The main issue with creating as many worker threads as there are physical CPU cores is that most likely the worker threads are underutilized, meaning that there's not enough work to fully saturate all workers most of the time.
Eventually, Windows will *park* CPU cores that it feels are underutilized. *Core Parking* is a power saving mechanism within Windows to not waster power on CPU cores that aren't doing anything. While in theory this is a good thing, the parking and unparking of cores does come with a slight performance hit, especially if this is done *over and over* again within a single frame.

Referencing the BSC talk, the effect of this can be seen in this graph:
![Too many cores, too little work](/assets/img/posts/dont_fight_your_os/core_parking_issue.png)
*Too many cores, too little work*

You can see in this graph that there are quite a few bubbles between individual jobs. Each of these bubbles is a possibility for the OS to go "oh some of the active cores are awfully underutilized, better park them", which I'm pretty sure does account for at least some of the bubbles seen in this graph.

The presentation even acknowledged that reducing the number of workers to 8 does increase the overall frame performance.
![Less cores, better performance?!](/assets/img/posts/dont_fight_your_os/core_parking_issue_2.png)
*Less cores, better performance?!*

During this part of the presentation 
