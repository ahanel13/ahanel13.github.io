---
title: "Understanding CL.0 and 0.CL Attacks: A Comparative Guide"
date: 2024-12-09 18:00:00 -0700 # 2024-11-29 19:00:00 -0700 # 7:00 PM
author: anthony   
categories: [Vulnerability Analysis, Request Smuggling]
tags: [request smuggling, security testing, burp suite, how-to]    # TAG names should always be lowercase
# media_subpath: /assets/posts/debugging-burp-suite-extensions
# image:
#   path: attachDebugger_dark.png
#   alt: Attach Debugger Icon
description: A guide to understanding and differentiating between CL.0 and 0.CL request smuggling vulnerabilities,
 with a practical analysis of testing methodologies.
---

I recently reviewed a reported issue regarding 0.CL. While reviewing issues isn’t part of my daily responsibilities, it’s something I encounter occasionally. The process can be challenging—understanding the testing methods, validating their legitimacy, and determining whether the findings pose a real security risk requires both knowledge and precision.

In this particular case, the report claimed a **0.CL Request Smuggling** issue had been identified. However, inconsistencies in the report stood out immediately. Part of the confusion was my own: I initially thought the issue was **CL.0 Request Smuggling**, a vulnerability I was very familiar with. However, once I realized my mistake, I came to the realization that I was only vaguely familiar with **0.CL Request Smuggling**. 

After some research and converstaions with others in the community, I want to clarify my understanding—and potentially help others—I’ll break down the differences between Request Smuggling types and the specific discrepancies between **CL.0 vs. 0.CL**.

---------------------------------------------------------------------------------------------------
## What is Request Smuggling 
Request Smuggling is a vulnerability that exploits inconsistencies in how a server or servers handle HTTP requests. These attacks manipulate the communication between servers, and sometimes clients, potentially allowing attackers to hijack sessions, expose sensitive data, or execute unauthorized actions.

### Server-Side Request Smuggling
Server-Side Request Smuggling occurs when differences in request parsing between two backend systems (such as a proxy and an application server) are exploited. For example:

- **Targets**: API gateways, load balancers, and reverse proxies.
- **Delivery Methods**: Sending maliciously crafted HTTP headers (Content-Length vs. Transfer-Encoding).
- **Goal**: Hijack connections, perform cache poisoning, or bypass authentication.
For more details, refer to PortSwigger's [HTTP Request Smuggling](https://portswigger.net/web-security/request-smuggling) documentation.

### Client-Side Request Smuggling
Client-Side Request Smuggling, also known as desync attacks, focuses on discrepancies in _browser-to-server_ communication. These attacks exploit misconfigured/misguided request handling between the browser and application server to enable same-origin bypasses, credential theft, or cross-user attacks.

For an overview, James Kettle’s research on [browser-powered desync attacks](https://portswigger.net/research/browser-powered-desync-attacks) is an excellent resource.

---------------------------------------------------------------------------------------------------

## What is CL.0 Request Smuggling
CL.0 is a server-side vulnerability caused by a discrepancy between how two backend servers interpret an HTTP request.

### How does this work?
The vulnerability exists between backend servers (Application servers behind a proxy/load balancer).
The victim/target could be other users on the platform or restricted applications behind a firewall.
Delivery Methods: Carefully crafted HTTP requests with ambiguous `Content-Length` or `Transfer-Encoding` headers.

### How to verify if it's valid issue
todo: 

### How to determine if issue false positive
todo: 

## What is 0.CL Request Smuggling
0.CL is a client-side vulnerability stemming from discrepancies between the browser (client) and the application server's request handling.

### How does this work?
- Who are the targets for this attack 
- What possible delivery methods.

### How to verify if it's valid issue
todo: 

### How to determine if issue false positive
todo: 

---------------------------------------------------------------------------------------------------

## So what happened to the issue?
Now that we've got a good background of the issue being reivew. This was the testing methodology, i.e. steps required to reproduce the issue under review.

1. Add the following HTTP request to a Repeater tab in Burp Suite:

> *Identifying information has been removed*
{: .prompt-warning} 

```http
POST /someEndpoint HTTP/1.1
Host: public.facing.host.com
Content-Type: x-www-form-urlencoded
Content-Length: 3
Connection: Keep-Alive

x=123
```

2. Disable Burp Suite's automatic Content-Length updating.
3. Create an additional `GET` request for the home page:

```http
GET / HTTP/1.1
Host: public.facing.host.com
Connection: Keep-Alive

```

4. Group the two requests into a single sequence in Burp Suite’s Repeater.
5. Configure the settings to send the group in sequence.
6. Hit Send and review the response for the second request

### Methodolgy Anaylsis
The testing methodology relied on manual manipulation of the `Content-Length` header, which can lead to unintended behavior in Burp Suite. This introduces potential false positives if not carefully managed.

Ultimately, the issue was invalid. The tester misinterpreted Burp Suite's behavior when manually setting the `Content-Length` header, mistakenly identifying a 0.CL vulnerability.


--------------------------------------------------------------------------------------------------------------
## Takeaways
While CL.0 and 0.CL share some underlying principles, their attack vectors and potential impacts are distinct:

- **CL.0**: Server-to-server discrepancies leading to backend exploitation.
- **0.CL**: Browser-to-server discrepancies enabling client-side exploitation.
Understanding these differences is crucial for accurate issue verification and mitigation.

> Futher Reading:
> - [https://portswigger.net/web-security/request-smuggling/browser/cl-0]
> - [https://portswigger.net/research/browser-powered-desync-attacks]
> 
{: .prompt-tip}
