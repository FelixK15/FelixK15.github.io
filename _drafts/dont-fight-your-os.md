---
layout:   post
title:    "Don't fight your OS - how to optimize multithreading workload"
tags:     programming c optimization
category: Programming
---

## Intro
This blog post was insipred by watching [Dennis Gustafsson's 2025 BSC Talk "Parallelizing the physics solver"](https://www.youtube.com/watch?v=Kvsvd67XUKw). You don't have to watch the video to follow the content here - In short, he describes the challenges he faced with utilizing multithreading to speed up the physics solver of the Windows PC version of his wonderful game [Teardown](https://teardowngame.com/).

While he does correctly identify the performance pitfalls with his jobsystem implementation, I feel like his solution could benefit from a different approach of how to overcome the performance problems outlined in his talk. And just to be perfectly clear, I'm only taking this talk as an example because of how recent it was held. Many games I've looked at as part of my role at Intel had the performance problems outlined in Dennis' talk and I wrote code myself that looked very similar to what Dennis presented.

So I want to use this blogpost to outline the typical pitfalls with common jobsystem implementations and how to get better performance out of your jobsystem.

> *Note:* This post will focus on building a job system for a PC game running under Windows. I'm too inexperienced with Linux to comment on how Linux works internally and I'm pretty sure this isn't relevant to game console development.

### What *is* a job system?
Before we start to talk about the pitfalls, it's important to classify my world-view about what a job system is.

Basically, it's a way to distribute self-contained *jobs* to several *workers* - each worker is typically run within a thread so that all workers can run in parallel on a given set of jobs.

If you consider this graph:
![A 1000-feet view of a typical job system](/assets/img/posts/dont_fight_your_os/job_system.png)

This is a 1000-feet view of a typical graph of modern games. You have a main thread, that start to do some sequential work (eg: read player input, send network packets etc), followed by a phase of parallel work that is being worked on by all available workers (eg: Entity-Component updates) and then followed by another sequential work phase (eg: send graphics commands to GPU).

Typically, you want all workers to be finished before you start the sequential phase followed by the parallel phase where all workers are utilized. So it's necessary to serve jobs to the workers and to have a mechanism that tells the main thread when all workers are finished.

### All your cores are belong to us
![Cats is way too greedy](/assets/img/posts/dont_fight_your_os/all_your_cores.jpg)

Most job systems I've seen are too greedy when it comes to consider how many threads to spawn for the job system (I'm guilty of this myself in one of the job systems I wrote in the past for the reasons listed below).

At it's core, many implementations look like this:

```c
JobSystem* pJobSystem = createJobSystem();
const int workerThreadCount = countLogicalCores() - 1;
pJobSystem->spawnWorkerThreads(workerThreadCount);
```

where `JobSystem::spawnWorkerThreads()` will spawn *N* threads either using [CreateThread()](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread), [pthreads](https://www.cs.cmu.edu/afs/cs/academic/class/15492-f07/www/pthreads.html), [std::thread](https://en.cppreference.com/w/cpp/thread/thread.html) or other APIs and `countLogicalCores()` will count the number of available logical cores of your CPU, as implemented in the following listing.

```c
int countLogicalCores()
{
  DWORD logicalProcessorInformationBufferSizeInBytes = 0;
  GetLogicalProcessorInformationEx(RelationProcessorCore, nullptr, &logicalProcessorInformationBufferSizeInBytes);
  char* pProcessorInformationBuffer = (char*)_alloca(logicalProcessorInformationBufferSizeInBytes);
  GetLogicalProcessorInformationEx(RelationProcessorCore, (PSYSTEM_LOGICAL_PROCESSOR_INFORMATION_EX)pProcessorInformationBuffer, &logicalProcessorInformationBufferSizeInBytes);
  PSYSTEM_LOGICAL_PROCESSOR_INFORMATION_EX pCurrentProcessorInformation = (PSYSTEM_LOGICAL_PROCESSOR_INFORMATION_EX)pProcessorInformationBuffer;
  
  int logicalCoreCount = 0;
  DWORD processorInformationBufferOffset = 0;
  while(processorInformationBufferOffset < logicalProcessorInformationBufferSizeInBytes) {
    for(DWORD groupIndex = 0; groupIndex < pCurrentProcessorInformation->Processor.GroupCount; ++groupIndex) {
      logicalCoreCount += (int)__popcnt64(pCurrentProcessorInformation->Processor.GroupMask[groupIndex].Mask);
    }
    processorInformationBufferOffset += pCurrentProcessorInformation->Size;
    pCurrentProcessorInformation = (PSYSTEM_LOGICAL_PROCESSOR_INFORMATION_EX)(pProcessorInformationBuffer + processorInformationBufferOffset);
  }
  return logicalCoreCount;
}
```

If you think about it, this makes perfect sense at first. This nicely scales your job system with the number of available cores, with the idea being that each worker will run on it's own core, thus maximizing thread utilization, which should result in the best perf possible - Commit-and-done!

...

Now, unfortunately the reality isn't that simple and the OS will most likey get in the way of your ideal world in the name of efficiency!

### Core parking
