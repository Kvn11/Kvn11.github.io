---
title: IcedId Malware config extraction
description: Unpacking and analyzing a 2022 Gozi malware sample
date: 2023-12-01 02:45:01-0700
image: img/cover.jpg
categories:
    - Malware
tags:
    - malware
    - reversing
    - windows
---

## Sample Info

[0581f0bf260a11a5662d58b99a82ec756c9365613833bce8f102ec1235a7d4f7](https://bazaar.abuse.ch/sample/0581f0bf260a11a5662d58b99a82ec756c9365613833bce8f102ec1235a7d4f7/)

## Initial Analysis

Based on the few number of imports, and the size of the `.data` section, this sample is likely packed.

![Why so little imports?](img/1.png) ![Sus data](img/2.png) 

## Unpacking

Unpacking proved to be more difficult than expected.
None of my breakpoints every got reached because my debugger exited before ever reaching the sample's entry point.
Additionally, after `x64dbg`'s call to `LoadLibraryW` to load the sample, I could see that the sample had some memory reserved, but none of it's sections were mapped.
I figured it was probably receiving the wrong reason from the calling process and this was an anti analysis feature.
But then after looking at `PE-Bear` a lil more I realized that this `DLL` has a number of exports, so I could just use `rundll.exe` to call the functions manually and pass this obstacle.

![Exported functions from the Dll](img/3.png)

After the dll is loaded in `x64dbg`, I just set breakpoints on each of the exported functions.

![Breakpoints](img/4.png)

When continuing, the first breakpoints I hit is the first function I called, `DllRegisterServer`.
This function was filled with instructions like `cmp al, al; jne` which were interesting because I don't think the jump would ever trigger.
My thought is that this was supposed to be an anti analysis technique by the authors but I am not sure.
Also, I quickly realized that debugging without knowing what I was looking for would be confusing and not very fruitful so I went back to binja to do some static analysis.
However, due to the fake branching, the code is a bit convuluted and hard to follow.
I decided to make a script to get rid of them.

```python
def remove_fake_branches(fn_addr):
    def is_jump(instr):
    if instr.tokens[0].text in ["je", "jne", "jz", "jnz"]:
        return True
    else:
        return False

def find_fake_jmps(basic_block):
    fake_jmps = []
    instructions = basic_block.get_disassembly_text()
    ctr = 0
    while ctr < len(instructions):
        instr = instructions[ctr]
        if is_fake_cmp(instr):
            #print(f"[i] Fake cmp @ 0x{instr.address:016X}")
            next_instr = instructions[ctr + 1]
            if is_jump(next_instr):
                print(f"[*] FOUND FAKE jump @ 0x{next_instr.address:016X}")
                fake_jmps.append( (instr, next_instr) )
                ctr += 1
        ctr += 1
    return fake_jmps

def patch_fake_jmp(instr_pair):
    cmp_instr = instr_pair[0]
    jmp_instr = instr_pair[1]
    jmp_type = jmp_instr.tokens[0].text

    if jmp_type == "je":
        bv.always_branch(jmp_instr.address)
    elif jmp_type == "jne":
        bv.never_branch(jmp_instr.address)
    elif jmp_type == "jz":
        bv.always_branch(jmp_instr.address)
    elif jmp_type == "jnz":
        bv.never_branch(jmp_instr)
    else:
        print(f"[!] Unhandled jmp @ 0x{jmp_instr.address:016X}")
        return False
    bv.convert_to_nop(cmp_instr.address)
    return True

def remove_fake_branches(fn_addr):
    n_patched = 0
    f = bv.get_function_at(fn_addr)
    for basic_block in f.basic_blocks:
        fake_jmps = find_fake_jmps(basic_block)
        for fj in fake_jmps:
            if patch_fake_jmp(fj):
                n_patched +=1

bv.begin_undo_actions()
remove_fake_branches(0x7ffdb45a1054)
bv.commit_undo_actions()
```
Also, you will need to disable `Tail Call Analysis` in whatever tool you are using, otherwise, there the psuedo c view will not be as concise as it could be.
And just like that, the control flow becomes so much easier to read.
Ngl, seeing this happen in real time was very satisfying.

![Before](img/5.png) ![After](img/6.png)

If we follow the function calls in this now super flattened function, we end up at what looks to be a function that does some api hashing, and uses stack strings.
I set breakpoints at these calls so I could figure out what api it was grabbing. 

![Main code](img/7.png) ![Looks like it wants to do something with a window?](img/8.png)

I also found a check where the malware will only run its attack if the year is `2022`.
I ended up patching this instruction to always jump in `x64dbg`.
I also had to change my binja theme back to regular dark because the struct member name color was the same color as the background, so it wasn't showing up in screenshots :/
I'll fix the theme a eventually, for now I'll just have to keep it.

`EnumWindows` accepts a callback function, which is a prime candidate for inserting a malicous function, so I decided to explore that next.

## Callback function

Again, this function has a lot of fake branches, so prune those first.
I then also read this [article](https://binary.ninja/2023/11/13/obfuscation-flare-on.html) that explain what tail calls were, and how to fix them, so I had to reopen my file with the correct settings disabled to make everything easier to read.

The callback function seems to just get a handle to a windows, then it enters a function that seems to be doing some type of memory copying operations.

![Maybe COM injection?](img/9.png) ![Maybe this is the unpacking routine?](img/10.png)

Following along some of the function calls results in finding what seems to be a promising function.
It seems to create a string `|SPL|`
Doing some OSINT reveals that this might be an IOC for `SplPacker`.

![ What is SPL?](img/11.png)