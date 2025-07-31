---
layout:   post
title:    "Typical job system pitfalls"
tags:     programming c optimization
category: Programming
---

### CPU Cores
Before we start to talk about the software side-of-things, we have to talk about the hardware. More precisely, about the CPU that was used as the basis for the performance metrics mentioned in the talk and why that matters.

The performance measurements that are shown during that presentation (and which I'll reference in this blog post) were taken on an **i9-14900F Intel CPU**, which is a hybrid CPU. In practise, this means that the CPU has 2 different cores that it's using to execute instructions. There are **P**erformance cores and **E**fficience cores (commonly referred to as P-& E-Cores), with the general, overdramatic, consensus being that P-Cores are fast and awesome and E-Cores are slow and lame. While the E-Cores *are* slower, it's not "half of the performance of the P-Cores", as was stated during the talk.

In fact, looking at [the specs for that particular CPU](https://www.intel.de/content/www/de/de/products/sku/236853/intel-core-i9-processor-14900f-36m-cache-up-to-5-80-ghz/specifications.html), in terms of base clock speed, the **E-Cores are around 75% as fast as the P-Cores**.

For completeness, we also have to consider the cache-setup of P-/ and E-Cores since that will help us later on to understand how threads are scheduled on hybrid-CPUs.
![i9-14900F cache setup](/assets/img/posts/dont_fight_your_os/hybrid_cache_setup.png) *i9-14900F cache setup*

From this poorly structured image we can see that, for an i9-14900F, **each P-Core has 80KB of L1**, which is little bit less than the **E-Cores 96KB of L1**.
**An E-Core cluster, consisting of 4 E-Cores, shares one 4MB L2 Cache** where as **each P-Core has it's own 2MB L2 Cache**. Finally, **all cores share a 36MB L3 cache**.

The Windows scheduler assigns threads to run on CPU cores. Ideally, we want threads to run on P-Cores, if they are available.
On Intel CPUs, the following priority system decides which CPU core to run a thread on:

1. Run on P-Core, if not available
2. Run on a single E-Core within an E-Core cluster (to minimize the contention on the shared L2 cache of the cluster), if not available
3. Run on an E-Core within the lowest utilized E-Core cluster, if not available
4. Run on P-Core SMT.

Now that we've learned about the hardware, we can talk about the software.


> *Note:* This post will focus on building a job system for a PC game running under Windows. I'm too inexperienced with Linux to comment on how Linux works internally and I'm pretty sure everything discussed in this blogpost isn't relevant to game console development in general.



Eventually, Windows will *park* CPU cores that it feels are underutilized. *Core Parking* is a power saving mechanism within Windows to not waster power on CPU cores that aren't doing anything. While in theory this is a good thing, the parking and unparking of cores does come with a slight performance hit, especially if this is done *over and over* again within a single frame.

> *Note: Core Parking* is an optional feature which is turned on by default for the **balanced** & **power saver** power plans in Windows. I've seen many threads in the Steam Community, Reddit, etc suggesting to turn this off but if you want to ship a game on Windows I'd assume only the enthusiast go the "extra mile" of actually disabling this, meanining that you still unfortunately have to account for this. The Laptop and Windows handheld gamers will thank you for it though ;)
