---
title: HTB Insane Web OOPArtDB
description: Hardest challenge on HTB
date: 2024-02-20 02:45:01-0700
image: img/hackthebox.jpg
categories:
    - HackTheBox
tags:
    - web
    - htb
    - insane
---

## Intro

This challenge allows you to submit a URL to be "scanned" by a bot on the challenge server.
The flag is shown in the webapp once you have successfully accessed the appropriate page with the credentials of the "Overseer" user.

## Foothold

One of the first things I noticed in the source code, was the `/debug` and `/register` endpoint.
The `/debug` endpoint caught my attention because it could reveal the secret `REFERRAL_TOKEN` value which is required to register via `/register` as the higher privilege `researcher` type of user.

```js
// challenge/index.js

app.get("/debug", util.isLocalhost, (req, res) => {
    let utils = require("util");
    res.end(
        Object.getOwnPropertyNames(global)
        .map(n => `${n}:\n${utils.inspect(global[n])}`)
        .join("\n\n")
    );
});
```

When accessed via a GET request from localhost, it responds with a detailed inspection of all properties of the global object in Node.js.
It's a way to inspect the global environment for debugging purposes.
In all honesty, at this point I couldn't see how I could leverage this to get the flag, but it seemed like a good step up since I was obtaining higher permissions.
This is probably I need to work on for the future.

