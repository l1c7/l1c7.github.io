---
title: 'HackTheSystem: JinjaCare'
author: lict
categories: [HackTheBox]
tags: [web, SSTI]
render_with_liquid: false
media_subpath: /images/hackthesystem_jinjacare/
---

The challenge description:
```
Jinjacare is a web application designed to help citizens manage and access their COVID-19 vaccination records. The platform allows users to store their vaccination history and generate digital certificates. They've asked you to hunt for any potential security issues in their application and retrieve the flag stored in their site.
```

The name of the challenge suggests SSTI with the following syntax: `{{ 7*7 }}`.
Let's make an account on the target website and see if we confirm the SSTI.

![20250629163757.png](20250629163757.png)

Tried adding a new vaccination record:

![20250629163912.png](20250629163912.png)
![20250629163944.png](20250629163944.png)

As you can see it did not work, let's move on and make a medical history:

![20250629164020.png](20250629164020.png)
![20250629164031.png](20250629164031.png)

It did not work too, let's see if we can insert the payload in `Personal Info`:

![20250629164134.png](20250629164134.png)

The website accepted the payload as a valid input, and it's already a good sign! Let's navigate to dashboard and download our certificate.

![20250629164300.png](20250629164300.png)

As we can see, it returned `49` as the name value, which means our SSTI payload worked!
We can achieve RCE and grab the flag using the following payload: `{{ self.__init__.__globals__.__builtins__.__import__('os').popen('cat ../flag.txt').read() }}`.

Inject it:

![20250629164643.png](20250629164643.png)

Download the certificate from this page:

![20250629164701.png](20250629164701.png)

Grab the flag:

![20250629164728.png](20250629164728.png)

That's it! The challenge was really easy. See the section below to learn more about the technique and resources used.

# Resources

- https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/Python.md#jinja2---remote-command-execution
- https://hacktricks.boitatech.com.br/pentesting-web/ssti-server-side-template-injection
