---
layout:   post
title:    "I reverse engineered vswhere so you don't have to"
tags:     programming
category: Programming
---

![Thumbnail](/assets/img/posts/vswhere_reverese_engineering/thumbnail.png)

I'm currently prototyping a Windows C/C++ build system for my personal toy projects with the goal of having a very small, self contained and portable build system that I can quickly add to projects. The build system reads one or more configuration files, resolve some paths and call the various compiler/linker executables - pretty standard stuff.

I don't want to go into too much detail about the build system in this post, but **I do want to talk about a problem that probably everybody ran into who built a build systems for Windows applications before**. The problem: How to get the path to cl.exe and link.exe?.

Since Visual Studio 2017, Microsoft ships *vswhere* for exactly this. For those of you, who don't know what *vswhere* is: ***vswhere* a standalone commandline tool that you can use to get the path to all your local Visual Studio installations**. In case of multiple Visual Studio installation (eg: if you have Visual Studio 2017 and Visual Studio 2022 installed), you can even supply a filter and tell it to only return the instance of Visual Studio that matches your filter (eg: "only list the instances that I can built C# projects with").

Which sounds nice in theory, has been an annoyance in one way or another for most custom build systems. While *vswhere* is part of any modern Visual Studio installation and will be installed in a fixed path (`%ProgramFiles(x86)%\Microsoft Visual Studio\Installer\vswhere.exe`), you still have to basically create a new process, pipe your arguments to that process, read its output and destroy the process again. **To me, that sounds like a lot of overhead to just get a single path**. So why can't we just figure out whatever magic *vswhere* is doing and just...do the same? 

*vswhere* is [open-source](https://github.com/microsoft/vswhere) after all so this should be easy, right?! 

> *Note:* To be fair, most build systems will probably cache the paths that *vswhere* returns, so theoretically the cost of retrieving the Visual Studio path is only paid once.

**If you're just interested in the result and not how I got there, feel free to jump ahead to the [TLDR](#tldr---whats-happening)**

## Inside vswhere's source code
We want to know: where does vswhere actually get its list of Visual Studio instances?

Looking at the source code, we can see that *vswhere* is a C++ library. It has a bunch of utility classes, but the meat is inside the [InstanceSelector class](https://github.com/microsoft/vswhere/blob/main/src/vswhere.lib/InstanceSelector.cpp).
Eventually, *vswhere* will call into [`InstanceSelector::Select()`](https://github.com/microsoft/vswhere/blob/main/src/vswhere.lib/InstanceSelector.cpp#L100), which will return a Visual Studio `SetupInstance`. You'll note here that *vswhere* differentiates between *Legacy Instances* and "normal" setup instances. Legacy instances are any Visual Studio installations before Visual Studio 2017.

> *Note:* Finding out where Visual Studio is installed before Visual Studio 2017 was rather trivial. On a 64-Bit Windows you could simply query the `HKLM\SOFTWARE\WOW6432Node\Microsoft\VisualStudio\SxS\VS7` reg key and on a 32-Bit Windows the `HKLM\SOFTWARE\Microsoft\VisualStudio\SxS\VS7` key. This is reflected by vswhere's [LegacyProvider class](https://github.com/microsoft/vswhere/blob/main/src/vswhere.lib/LegacyProvider.h) which will do exactly that.

For non-legacy instances, it'll use the supplied `IEnumSetupInstances` pointer to call into `IEnumSetupInstances::Next()` to get an `ISetupInstancePtr` object to be used later to get the installation path - so at this point it has already a list of all the setup instances. **Damn - we have to go deeper!**

### Where do all the instances come from?
Looking at where `InstanceSelector::Select()` is [called](https://github.com/microsoft/vswhere/blob/main/src/vswhere/Program.cpp#L68), we can see that the `IEnumSetupInstances` object that the InstanceSelector is using is returned by the [`GetEnumerator()`](https://github.com/microsoft/vswhere/blob/main/src/vswhere/Program.cpp#L137) function. It'll use a previously created `ISetupConfigurationPtr` object and cast it to a `ISetupConfiguration2Ptr` using [`IUnknown::QueryInterface()`](https://learn.microsoft.com/en-us/windows/win32/api/unknwn/nf-unknwn-iunknown-queryinterface(q)) and finally call `ISetupConfiguration2Ptr::EnumAllInstances()`. **Big Sigh** Let's take a quick break - this has C++ CLI/.NET mumbo jumbo all written over it. I was naive enough and hoped it'd just be a simple C++ library. Silly me!

Unfortunately we are already at the end of the usefulness of the *vswhere* sourcecode, because `ISetupConfiguration` and its greatly named sibling, `ISetupConfiguration2`, are not part of the publicly released sourcecode - **BUMMER!**.

Where do we go from here?

If those classes are not part of *vswhere*, surely they must be coming from somewhere?

Inspecting *vswhere.exe* inside [Dependency Walker](https://www.dependencywalker.com/) doesn't yield any results, but **looking at [*vswhere*'s .packages file](https://github.com/microsoft/vswhere/blob/main/src/vswhere.lib/packages.config), we can see that it's referencing the suspiciously named *Microsoft.VisualStudio.Setup.Configuration.Native* package.** Unfortunately though, this package is not open source, **so we'll have to resort to reverse-engineering**. Searching for the name of that package on my local machine pointed me to `%ProgramData\Microsoft\VisualStudio\Setup\x64\Microsoft.VisualStudio.Setup.Configuration.Native.dll`. I decided to load that dll into [*Ghidra*](https://ghidralite.com/) and see what I can find.

Time to look at some binaries...

### Reverse engineering - strings are your friends
After opening the dll in *Ghidra*, I was again remembered that the code inside here are some C++ CLI .NET classes - I hoped to find a big list of exported functions with clear names that could point me to the right direction, instead I was greeted by a tiny export list...

![List of exported functions](/assets/img/posts/vswhere_reverese_engineering/exported_functions.png) *List of exported function by Microsoft.VisualStudio.Setup.Configuration.Native.dll*

My hopes of finding anything useful inside `GetSetupConfiguration()` was crushed since it's just a wrapper around allocating a `SetupConfiguration` instance.

However, looking at the exported classes list, we find our old friend `SetupInstance` again.

![SetupInstance in class view](/assets/img/posts/vswhere_reverese_engineering/setup_instance_class_view.png) *SetupInstance in class view*

From there we can find all the places in the code where `SetupInstance` is getting instantiated. While I did follow this lead for some time, I didn't come to a good conclusion unfortunately.

To try a different approach, I like to search for known strings inside the binary I'm reverse engineering. **Searching for known strings inside a binary will more often than not get you close to the code you want to reverse engineer**. And while I didn't have a particular string in mind to check against, **I just searched for "VisualStudio" in the hopes of finding anything that'll get me on the right track**.

![Result of "VisualStudio" string search result](/assets/img/posts/vswhere_reverese_engineering/string_search.png) *Result of the string search for "VisualStudio"*

I first followed the first two entries since they look like registry keys...**It couldn't be as easy as just querying the registry, could it?!**
Unfortunately not. The registry keys are used to search for the Visual Studio installation path. 

Then, with the third entry I struck gold. The core around `%ProgramData%\Microsoft\VisualStudio\Packages` is exactly what I was looking for.

>*Note:* Many functions I've inspected in Ghidra seemed to handle `char`<->`wchar_t` conversion. *vswhere* seemed to do a lot of allocations and copies behind the back to handles these conversions. If you don't need that for your project, you can already save a lot of overhead here. Might not be much on a per-build basis but running your build a thousand times will add up eventually!

### TLDR - What's happening?
*vswhere* will search the following registry keys for the `CachePath` value:
  - `HKLM\SOFTWARE\Policies\Microsoft\VisualStudio\Setup`
  - `HKLM\SOFTWARE\Microsoft\VisualStudio\Setup`
  - `HKLM\SOFTWARE\WOW6432Node\Microsoft\VisualStudio\Setup`

If it didn't find a matching key, it'll fall back to the `%ProgramData%\Microsoft\VisualStudio\Packages` path.
Inside that path, there's an `_Instances` folder which will include **one entry for every local Visual Studio installation**. Inside each "instance folder" is a `state.json` file which will contain information about what particular version of Visual Studio this instance represents, what packages it has (basically answering the question "can it build XYZ?") and, finally, **where this Visual Studio instance is installed**.

Excerpt of the information inside `state.json` that probably most users are interested in:
```json
 {
  "installationName": "VisualStudio/17.8.3+34330.188",
  "installationPath": "C:\\Program Files\\Microsoft Visual Studio\\2022\\Community",
  "selectedPackages": [
    {
      "id": "Microsoft.VisualStudio.Component.VC.Tools.x86.x64",
      "version": "17.8.34129.139",
      "type": "Component",
      "selectedState": "IndividuallySelected",
      "userSelectedState": "Explicit"
    }
  ]
 }
```
