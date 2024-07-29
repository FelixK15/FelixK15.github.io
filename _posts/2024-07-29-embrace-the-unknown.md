---
layout:   post
title:    "Embracing the unknown"
tags:     programming
category: Programming
---

## Intro
This is a reaction to [Sebastian Schöner's](https://blog.s-schoener.com/) blog post [Get Shit Done](https://blog.s-schoener.com/2024-07-27-get-shit-done/).
If you didn't read his blog post, be sure to take a second to read it, it isn't long, it's awesome and I'm going to touch on what he wrote.

I specifically want to expand on the bullet point "catch the ball and run with it" with something that I'd like to call "embracing the unknown"

### Embrace the unknown
I'm fortunate to have had the privilege of working for game development studios that utilize their own in-house technology. Working on custom technology while simultaneously shipping a game is *hard*. The closer you get towards release the more stress is being put on the custom tech and the more things tend to break/need to be touched. Naturally this meant that engine programmers in those studios had to wear a lot of different hats to fix the various problems at hand. Problems that reached from finding optimization opportunities to fixing bugs to adding tools to extending the build pipeline, etc. 

What I want to drive home with this awfully self-glorifying passage is that the environment I've been able to grow my career in naturally produced engineers who gladly took any ball thrown at them. Growing up in that environment meant that it became second nature to dive into codebases that I wasn't familiar with or to fix bugs/add features in a specific domain that I had very little experience in. Adopting this skill-set has become an invaluable asset and something that I've, unfortunately, not seen many other engineers do (especially outside gamedev).

Engineers who weren't exposed to this usually don't have the "catch the ball and run with it" mentality and usually block attempts of "passing the ball" with excuses like "I'm not familiar with this codebase", "not my responsibility" or "not that important". And while all of these statements have their right to exist, I think in general we can do better.

What I would like to see are engineers who aren't afraid to embrace the unknown and look deeper into a problem even if it's outside their domain knowledge and/or comfort zone. This could mean many things but to give you some pragmatic examples:

- If you have an error in your `std::vector<>` usage-code and don't know how it works under the hood? [Just take a look!](https://github.com/microsoft/STL/blob/8657d15b3ddfacb121e88e307f9a18b3fa379185/stl/inc/vector)[^1]

- If you want to know how C++ lambda captures work in clang? [Open godbolt and find out!](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGe1wAyeAyYAHI%2BAEaYxBIAnKQADqgKhE4MHt6%2BekkpjgJBIeEsUTFc8XaYDmlCBEzEBBk%2Bfly2mPZ5DDV1BAVhkdFxtrX1jVktCsM9wX3FA2UAlLaoXsTI7BzmAMzByN5YANQmm24T%2BKgAdAhH2CYaAIJbO3uYh8dUXgxVAmLXtw/3BAAnglMFgqPtTiAQO9Ph0jm4AG6oPDoCDza77JgKQGfABiHy%2BDAAKsDMEcrPcsTjkPjYWkSSD9jDCXdiMQmICTABWCxcDTcgAiryFJgA7BZ%2BaKBeS/sECEyCR1WezAR4PvKjkL%2BZsKXckSj9phVJUvARMAK2hzQRAqXjFfTSQq6QJ0fcxbr9p6nSy2RzuRZmUrfarloJrOGuSLNkLA2kZW6pX9ZYJ9iwmMEIHLMcRgMhSPtkAg6gAqYvZ4AI10PcV/L37LOiBIEFaggCSKc1%2B21Hq9RpNZotBkB1v95gAbIK0e7a3WvQpRAwqBBzGZzFz0Cv8%2BPG83iG3BOidTPZ/s97uGPG7nWxQLD7rj0ySJmU3hhV3yfXXm5vUGVWqwzqhyWNYeBVteNb3CenqxgIyp%2BjyeCTneD43kmkFemeKwMO%2BR4JgKHCLLQnBcrwfjcLwqCcG44aWBCyyrC8Ww8KQBCaARpAQEgaBYLghAkOQlA0PQzBsJwzH8IIIhiOwUgyIIigqOoHBaKQuhmPohjGDRNgVISLgMO4nhNCA6mBNMRQlCAmwaIkySpAIox%2BOpOT2QwvQWQM1mtO01STI5JneZUHRdPU7n9DEXkTN0/nqVFoXmeFVkaIsCj0WsnCPJ8zxfqcTiXL8boAqSYL1oIEDFraNL2gIDKYPMED6ugxbIZS2J2s6xKOjBDBwZyPJ8oKb7upK0q4XcWavp23bJvKfbIKamB3G1yA2sttKErVP5pPmjVlgkCnEAKTC1FW07odB1U9cG/qIcBwGRm%2B3WXnWmHENh014Wh40pnuTDoAd7YJKaDXIk1p0QVeXpZhEBhvh9kOevOhhLiua4bmY6lAWOMNMC1COlfKONmHDz0YZg577ETl6oYV33ymmGZZnUub5oWJZlszlZJhDdZzQtS3UhAv3/UoxCA6a%2BYMF4tC0E2xB4w%2B/DEM%2B8q0CTgFq/C9YftptBgV6Z343W3W9f6tCTlLMty3j4HSrTL3k1hOH3lK7FERwJGkGRKmURw1F3RYdErGsQGbGYvCscpBGLAA1lZXLnFysQABzJxokjJ5IY58psGf6Jwki8CwEgaDZ3sUZwvAKCANmR1oixwLAXGoCwCR0NEAkQGgrftzEuyaQA%2BgQxAfDHfB0GaxDVxAERsaQETBHUgJibwC/MMQgIAPIRNoQUr6Q3dsIIm8MLQy9R6QWARF4wBuGItDV%2BRl%2BYGmRjiBf%2BB7lUCKYI/Kl82afeco2hz1oHgCI7IN4eCwHPYeeBi5Px/sQCIyRzQv00mAowbFFhUAMMABQAA1PAmAADum8QRkXErIKS4hZISXkEoNQc9dAtAMFg0wAd9DgOrpARYqAmxpEfgAWk3uHUgqAkHEBRL/eAKU2hBTSPpQymRmikDMoURKLQXIdH8louyHQwqzFKIFQkIUGhGTGCY4KkxDGWXGH5Cxqi4pTA0UYiQKU0oyUIsRUic9fb7FUMnMcQixySALGw4A%2BwIDD1HvMKJvEiDEFDlweYEdsEcWbj3egZAKBdxbm3bJKAIlDxHgwMeQlJ7T1nhfNeS9961I3tvXeDh96H0YAQE%2BZ855XxvnfGWj9mJYFfsAd%2BKlP4KJ/n/XgAD1jMWAe7FSYCIFL2gesFScCEHMSQSgpQFphmYNAFHHBeDCHELIRQ/e9CaEyWkPQhSTCL66E2Bpdh2kuERB4WicRAiBDCNEfsIRVBHBsCEXuJI9QKKSOkZ8uRPlnAQFcLotRBlbFzFsrkNISLtFpFRcY3S1joqOL0Pi3y3RcXEocSoilZKEpuJSUsYO7AzDeI9r4i%2B/jAnBNCeEzSUSYllLiRABJJBQ7MrSUcjJKB8m907t3ApAxgBSBaBU6IVS54NPPsxDVTS95PzacfU%2B58xmYGvrfe%2BAzeBDIwWsy1eAv6OEmXPGZQDBAgIvksyBIYYEXw2fvbZqC9kYOCIc%2BufATlENIeQxglzqGiFobc2Q9ylIqTUi8rSnClkwu%2BR0R%2BkLohSKwFmkl8LEVEpaOomYdj0WuSxfonFtKq3Fs6JS4y4x5GmJsQ2tFzikXOPJfS1KjKTIss9uXcRnAAlBJCWE3MyB9hSHOFweJ%2BBEmitSSxbBsd47nA0GYaykhYibE2FyXOsRJAaC5PnDghdSDFz5GXPxldbA1w3RKxuiAQDLAIEDAgsrpXZNCKwdYnLp08qMHy0pMd12gj4gWvQVy403LkgwxSzC9AkPZAkFeI62U%2B04JvU0P79ioHBCB7l/dwPRMg4KjwWTojJPXXXdinFP2EYlrkuVvdAOiQ4GRsJFHIlUdidMldJAUQBQQ9JCQCb5KMOTToKypAMNMCw%2BRHDXtH0cAI9%2B00xHSNTvIxEiDsSom0flUkrYmxGObtIHHMwsRzibFFMnKQXIXPJy5JIUUXAw5XpvWO32VcX1MfmCysRAWn0hcWEglIzhJBAA%3D)

- Have a binary that behaves strange and you want to find out why? [Try to open the binary in Ghidra and see if you can make sense of it](https://ghidra-sre.org/)

Depending on the size of the ball played to you, this can be a huge return of investment because you always end up learning new things along the way that'll help with similar tasks in the future - a new tool in your toolbox, so to speak. I understand that leaving your area of expertise is scary and not having all the answers for the problem before you feels unproductive, but once you've broken that mental barrier, many good things will follow. 

> *Note*: Of course, there's a fine line between spending too much time investigating and consulting a domain expert and, if necessary, delegating the task. But at the very least you can help to narrow down the issue by forwarding what you've already looked into - and hey, you might still have learned something new for the next time!

Naturally, you can't fix every bug on the planet and eventually, if you're stuck or if you've spent too much time without getting a result, it's time to consult a domain expert. But even then it's still possible to embrace the unknown by preparing a list of questions that you can use to jump-start a 2nd attempt at fixing the problem without just handing it off and missing the opportunity to expand your knowledge in that particular domain.

These questions can range from 
- "Where would you suspect this bug?" to 
- "What would be the best place to put this feature in the codebase?" to 
- "Does my assumption of how this system works align with reality?" 

Don't be shy, if you're not super obnoxious, there's really nothing you can do wrong here :)

Stepping out of one's comfort zone is essential for growth. Embrace challenges, seek knowledge beyond familiar areas, and use every opportunity to learn and expand your expertise.

As a closing statement I'd say if you, so far, stayed in your comfort zone, be sure to leave it now and again - it's worth it!

------

[^1]: MSVC STL just as an example, other STL implementations are open source as well
