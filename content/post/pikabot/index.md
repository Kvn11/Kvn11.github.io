---
title: Pikabot malware analysis
description: Unpacking and analyzing a Pikabot malware sample
date: 2023-12-12 02:45:01-0700
image: img/cover.png
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

## Unpacking


