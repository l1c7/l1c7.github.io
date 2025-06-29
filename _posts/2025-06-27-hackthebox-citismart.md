---
title: 'HackTheSystem: CitiSmart'
author: lict
categories: [HackTheBox]
tags: [web, auth-bypass, weak-authentication, SSRF]
render_with_liquid: false
media_subpath: /images/hackthesystem_citismart/
---

The challenge description:
```
Citismart is an innovative Smart City monitoring platform aimed at detecting anomalies in public sector operations. We invite you to explore the application for any potential vulnerabilities and uncover the hidden flag within its depths.
```

For this challenge I couldn't make an account but we had a login form, which means we are supposed to hack it. I started analyzing `JS files` from `Debugger` to see if I can find something interest there, and I found this:

![20250629180855.png](20250629180855.png)

When I navigated to `/api/dashboard/endpoints`, it said `{"message":"cookie token is not found"}`, let's set one running the following command in the console: `document.cookie = "token=lictwasherelmfao; expires=Tue, 01 Jan 2026 12:00:00 UTC; path=/";`

Now we can access internal hosts:

![20250629181242.png](20250629181242.png)

Also, if we navigate to `http://94.237.57.115:34986/api/dashboard/metrics` we can see more detailed data about the endpoints we have in the list. Basically, the `/metrics` endpoint contains the data our server was able to get from internal hosts, making our `SSRF` non-blind.

Also, I accidentally discovered that there is a `/dashboard` endpoint that makes working with `endpoints` easier because it provides a `frontend` for all of the operations we can do:

![20250629181549.png](20250629181549.png)

As we can see, if we're trying to access an internal port that is closed, we're getting this error:

![20250629181707.png](20250629181707.png)

At this point, I vibe coded a tool to make a full port scan on `127.0.0.1` and discovered that the port `5984` is open. This port is reserved for `CouchDB`, so let's enumerate it by adding `127.0.0.1:5984/_all_dbs`:

![20250629181848.png](20250629181848.png)

Navigating to `http://94.237.57.115:34986/api/dashboard/metrics` we'll notice that we got the DB's name which is `citismart`:

![20250629181951.png](20250629181951.png)
Now, let's probe `http://127.0.0.1:5984/citismart/_all_docs?` to understand how to DB looks like:
![20250629182108.png](20250629182108.png)
![20250629182135.png](20250629182135.png)
That's it! We're one step away to get the flag. Simply add the `127.0.0.1:5984/citismart/FLAG?` endpoint, navigate to `/api/dashboard/metrics` and copy the flag!

![20250629182255.png](20250629182255.png)
![20250629182311.png](20250629182311.png)
