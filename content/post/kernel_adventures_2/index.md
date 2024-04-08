---
title: Intro To Kernel Exploitation
description: Learning kernel exploitation via ctf challenges.
date: 2024-04-08 02:45:01-0700
image: img/hackthebox.jpg
categories:
    - HackTheBox
tags:
    - pwn
    - htb
    - medium
---

## Setup

This is the solution for HTB's `Kernel Adventures: Part II`, and in the process, we can learn and get an introduction to kernel exploitation.
After you download the challenge, just follow the instructions in the README to get the required stuff installed.
However, you may get the errors:

```text
error: patch failed: Makefile:1115
error: Makefile: patch does not apply
error: patch failed: arch/x86/entry/syscalls/syscall_64.tbl:370
error: arch/x86/entry/syscalls/syscall_64.tbl: patch does not apply
error: patch failed: include/uapi/asm-generic/unistd.h:880
error: include/uapi/asm-generic/unistd.h: patch does not apply
```

during the `git apply` command.
To resolve these, you first need to run `git checkout ff1ffd71d5f0612cf194f5705c671d6b64bf5f91` to revert the repo to the commit in which the vulnerability was introduced.
Make sure the following packages are installed if you are using kali:

```
bison
flex
libelf-dev
bc
```

Then add the following options to the `make` command:

```bash
make -j $(nproc) CLFAGS="-Wno-error"
```

When you arrive at this choice, just select 0:

```
Initialize kernel stack variables at function entry

```

This choice determines whether to use a pattern to init variables on the stack, use 0, or do not auto init anything.
The purpose of using either 0 or a pattern is to eliminate all classes of unitialized stack variable exploits and info leaks.
We can use the first option for now.