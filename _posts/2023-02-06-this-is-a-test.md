---
layout: post
title:  "Writing native software for apple devices without using Objective-C"
---

## Summary
In this post I'll go into detail about how, during my contracting work for [Shimmer Industries](https://www.shimmerindustries.com/) (a company founded by Eskil Steenberg which is focused on developing a real-time lighting designing software) I worked on a piece of software which enabled us to use the MacOS and iOS APIs using a custom C API without having to use Objective-C.

## "Why would you do this? Just use Objective-C!!!11"
Let me start this post by saying that yes, we could've used Objective-C to be able to use the MacOS and iOS APIs (referred to as "Apple APIs" from this point forward) from within the C codebase that was already in place. We decided against it though, because we wanted to have a pure C codebase without any of the weird[^1] Objective-C code in there. We didn't want maintainers (who are all mostly familiar with C) to first learn how to parse Objective-C and also thought that this would be a cool thing that might also interesting other developers. It was certainly an interesting experience for me since outside some courses in University, I've never touched a OSX or iOS development and was also unfamiliar with Objective-C and XCode as an IDE.

## The plan

## Introducing the Objective-C runtime
Quoting the official documentation it states that "The Objective-C runtime is a runtime library that provides support for the dynamic properties of the Objective-C language"[^2]. This sounds interesting, but slighty vague, so after taking a closer look at the documentation and the API itself, to see if there's stuff in there that could be of use for solving this problem, I was delighted to see that there's stuff like "give me a list of all classes and methods" and, most importantly "call the function with a given name on this object". Bingo, this is exactly what I was looking for and seems to be a good foundation to build a C API on. Even better: The Objective-C runtime itself is even a pure C API! To test my working theory, I coded up this small simple sample to see if this worked.

```c
#include <objc/runtime.h>
const int totalClassCount = objc_getClassList( NULL, 0 );
const size_t totalClassBufferSizeInBytes = totalClassCount * sizeof( Class );

Class* ppClasses = ( Class* )malloc( totalClassBufferSizeInBytes );
objc_getClassList(ppClasses, totalClassCount);
```


So to first figure out how to approach this problem, it's imperative to understand how Objective-C code is getting compiled and what the resulting ASM code looks like when interacting with the Apple SDK. I figured it would just be a bunch of `call` instructions but read somewhere that Objective-C was behaving differently. To find out what was *actually* going on, I created a simple test project in XCode and looked at the disassembly and, while it was just a `bl` instruction (`branch and link` - basically `call` in the ARM world) for function calls, there was an odditiy in that the symbols for each of those `bl` adresses were `objc_msgSend$mainWindow` for a function call to `mainWindow()`


[^1]: Yes, I know that this is highly subjective.
[^2]: [https://developer.apple.com/documentation/objectivec/objective-c_runtime](https://developer.apple.com/documentation/objectivec/objective-c_runtime)
