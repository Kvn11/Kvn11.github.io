---
title: Mastering leetcode P1 - Topological Sort
description: Topological sort is one of the main patterns seen in most DS / Algo interviews. 
date: 2024-02-18 02:45:01-0700
image: https://th.bing.com/th/id/OIP.Jia0SYyUs0LPd97k4SKwbAHaEA?rs=1&pid=ImgDetMain
categories:
    - Algorithms
tags:
    - leetcode
    - interview prep
    - programming
---

## Intro

Back when I was studying for SWE interviews, there were 13 patterns that I read were prevalent in most interviews, and did encounter in my interviews.
Knowing them perfectly, and when to apply them was essential to passing.
One of the harder ones is topological sort.
This is a type of sorting algorithm used on graphs that are **Directed and Acyclic**.
This is because the algorithm will always return the nodes with no "children" last, and will print the parents first.
The examples on Geeks for Geeks is this:

![Directed Acyclic Graph](https://media.geeksforgeeks.org/wp-content/uploads/20231016113524/example.png)

The "parent" nodes will be printed first, so in this case it would be either 5 or 4 first, followed by 2, then 3, then 0 or 1.
Since its possible for there to be multiple "parent" nodes, then its possible for the final order to be expressed in several different ways.
However, there is a simple rule that must be followed.
Given two nodes (connected in 1 direction) ` Node 1 -> Node 2 `, `Node 1` must come first.

## Implementation

The actual implementation isn't super difficult, and its been covered extensively and better elsewhere so I won't go into super deep details.
First we need to define what a "parent" node is.
In this case, it would be a node with no `in-degrees`, which means that no nodes point to it.
In most of the problems you will be given, the input will usually be an array or list of directed edges, so as you build your graph you can also keep track of the degrees for each node, and then start the algorithm at the nodes with the least amount of degrees.
For every node with 0 in degrees, you remove that node and subtract the node from the in-degrees of its children.
Then repeat until finished.

Another approach is using recursion.
given any node, you can use DFS to travel to through its children until you reach a node with no children, and then you add that child to a stack, and then
add the parent to the stack.
You have to keep track of the visited nodes, but in the end you will have a correctly sorted stack.

The first approach will require a way to keep track the in-degrees of every node, and sort them.
The second approach uses recursion but doesn't require a method for keeping track of the in-degrees.
In both you need a way to keep track of visited nodes, and a stack or queue to print out the final result.

## Application

You generally want to apply this algo whenever you see a problem that has some type of dependency.
It's easier to explain with some examples.

## Example 1

> There is a new alien language that uses the English alphabet.
> However, the order of the letters is unknown to you.
> You are given a list of strings `words` from the alien language's dictionary.
> Now it is claimed that the strings in `words` are sorted lexicographically by the rules of this new language.
> If this claim is incorrect, and the given arrangment of string in `words` cannot correspond to any order of letters, return "".
> Otherwise return a string of the unique letters in the new alien language sorted in lexicographically increasing order by the new language's rules.
> If there are multiple solutions, return any of them.

### Test Case 1:

> Input: words = `["wrt", "wrf", "er", "ett", "rftt" ]`.
> Output: `"wertf"`

### Test Case 2:

> Input: words = `["z", "x"]`
> OUtput: `"zx"`

### Intuition

So here we have a list of strings, where letters are supposed to have "dependancies" (as implied by the use of the word lexicographically).
What does this mean?
Basically alphabetical order, but in whatever order the aliens use.
So the first part of the problem is to figure out the dependancies.



## Example 2

## Example 3