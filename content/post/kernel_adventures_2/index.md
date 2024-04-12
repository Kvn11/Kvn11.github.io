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
After you download the challenge, just follow the instructions in the README to get the required version of the linux kernel installed so we can take a look in our code editor.
However, you may get an error during the `git apply` command.
To resolve these, you first need to run `git checkout ff1ffd71d5f0612cf194f5705c671d6b64bf5f91` to revert the repo to the commit in which the vulnerability was introduced.
Now we have the code for the vulnerable kernel, and the author provided the image meant for the challenge.

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
Maybe we could just create new users until we overflow the `uid` value to be 0 again?
The the pointer to the child is copied into both the main list, and the current user's child list, and the `id` is incremented.