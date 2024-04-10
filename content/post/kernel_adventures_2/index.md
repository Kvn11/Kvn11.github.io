---
title: Intro To Kernel Exploitation
description: Learning kernel exploitation via ctf challenges.
date: 2024-04-08 02:45:01-0700
image: https://i.pinimg.com/originals/d8/73/c9/d873c94e242bafe6bbcfa83cde3b8b42.jpg
categories:
    - HackTheBox
tags:
    - pwn
    - htb
    - medium
---

## Setup

This is the solution for HTB's `Kernel Adventures: Part II`, and in the process, we can learn and get an introduction to kernel exploitation.
After you download the challenge, just follow the instructions in the README to get the required version of the linux kernel installed so we can take a look in our code editor.
However, you may get an error during the `git apply` command.
To resolve these, you first need to run `git checkout ff1ffd71d5f0612cf194f5705c671d6b64bf5f91` to revert the repo to the commit in which the vulnerability was introduced.
Now we have the code for the vulnerable kernel, and the author provided the image meant for the challenge.

## Finding Vulnerability

So my first approach here is to look at the `patch.diff` file to see what was changed.
There was a folder `magic/` that was added.
Also, there is a new syscall `magic` added to `arch/x86/entry/syscalls/syscall_64.tbl`
However, the "meat" of the challenge is in `magic/magic.c`, which fleshes out the syscall.

From just a quick read through the code, a couple idea's pop out.
The first one is that there is a condition where you can switch to a "child" user without a password being required.
Therefore an approach to consider is trying to get the root user to be added as a child of our current user.

```c
    // Try and switch to a child
    index = locate_user_by_name(me->children, CHILDLIST_SIZE, username);
    if (index == -1) {
        // Not a child, look for the user in the global list
        index = locate_user_by_name(magic_users, MAINLIST_SIZE, username);
        if (index == -1) {
            // User doesn't exist at all
            return -ENOENT;
        } else if (index == 0) {
            // Prevent logging back in as root
            return -EPERM;
        }
        child = magic_users[index];
        // Check the passed password is correct - if no password was passed, fail
        if (password == NULL) return -EFAULT;
        if (strncmp(password, child->password, 64) != 0) {
            return -EPERM;
        }
    } else {
        // Switching to a child is allowed without the password
        child = me->children[index];
```
