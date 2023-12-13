---
title: Pikabot malware analysis
description: Unpacking and analyzing a Pikabot malware sample
date: 2023-12-12 02:45:01-0700
image: img/cover.jpg
categories:
    - Malware
tags:
    - malware
    - reversing
    - windows
---

## Sample Info

[7e26c4f6f313e5248898a1dbe706ae5b998e12ff16947cb3bfda690ca62612c4](https://bazaar.abuse.ch/sample/7e26c4f6f313e5248898a1dbe706ae5b998e12ff16947cb3bfda690ca62612c4/)

## Initial Analysis

Based on low number of imports, plus a large and high entropy `.rsrc` section, this sample is likely packed.

![ Signs of packing ](img/1.png)

Among the imports are `IsDebuggerPresent` and `GetTickCount` which could be signs of anti-vm techniques that are present in this sample as well.
There were also a large number of exports, all named after some drawing related functionality.

## Unpacking

Due to the massive amount of exports, and not having access to the delivery method part of the sample, figuring out the function responsible for unpacking is going to be difficult.
However, Pikabot is known to use a lot of anti analysis, so a good place to start would be to look for usage of `IsDebuggerPresent`, since the locations in where it appears are likely part of the unpacking process.
I just hope every function doesn't contain it, but I think this method should work to narrow down the functions to look at first.

