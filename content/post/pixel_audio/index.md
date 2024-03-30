---
title: HTB PWN Pixel Audio
description: Solution for the HackTheBox Pwn Challenge Pixel Audio
date: 2024-03-30 02:45:01-0700
image: img/hackthebox.jpg
categories:
    - HackTheBox
tags:
    - pwn
    - htb
    - medium
---

> **Challenge Description**: One of our embedded devices has been compromised.
> It was flashing a message on the debug matrix that was too fast to read, although we managed to capture one iteration of it.
> We must find out what was displayed.
> To help you with your mission, we will also provide you with the fabrication files of the PCB module the matrix was on.

## Walkthrough

The zip files contains a Gerber module, as well as a `.csv` containing what seems to be the outputs of different `GPIO` pins, as well as the time they were recorded.

Having never really worked with a Gerber module before I had to do research to figure out its importance and usage.

Gerber describes the elements of a printed circuit board (PCB).
It is used for both the fabrication of the board, and its assembly.
A quick google search tells me that I can open the files using `gerbv`.
For some reason, the window would go black whenever I tried importing the `.DRL` files, so I excluded those. However, this is the final result:

![Gerber File](img/img_1.png)

There is a term `Common Anode Matrix` that I have never heard of before, as well as the MCU on the board being a `Raspberry Pi 3b+ Hat`.
ChatGPT reveals that a common anode matrix is simply a grid of LED's, and specific LED's are supplied voltage to create images, letters, or numbers.
Based on this, I can assume that the flag is flashing on the matrix, and I need to map the outputs of the `.csv` file to its LED on the board.
I was able to find the following image online which nicely displays the pinout of the Raspberry Pi:

![Pinout](img/img_2.png)
