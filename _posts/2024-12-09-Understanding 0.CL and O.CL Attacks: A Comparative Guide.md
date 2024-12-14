---
title: "Understanding CL.0 and 0.CL Attacks: A Comparative Guide"
date: 2024-12-09 18:00:00 -0700 # 2024-11-29 19:00:00 -0700 # 7:00 PM
author: anthony
categories: [Vulnerability Analysis, Request Smuggling]
tags: [request smuggling, security testing, burp suite, how-to]    # TAG names should always be lowercase
media_subpath: /assets/posts/understanding-cl-0-and-0-cl-attacks
image:
  path: CL.0 vs. 0.CL.png
description: A guide to understanding and differentiating between CL.0 and 0.CL Request Smuggling vulnerabilities,
 with a practical analysis of testing methodologies.
---

<style>
  <!-- Table override to prevent horizontal scrolling -->
  .table-wrapper {
    overflow-x: clip;
  }

  .table-wrapper td:nth-child(1) {
    white-space: nowrap;
  }

  .table-wrapper td:nth-child(2) {
    white-space: normal;
  }
</style>

I recently reviewed a reported issue regarding 0.CL. While reviewing issues isn't part of my daily responsibilities, it's something I encounter occasionally. The process can be challenging—understanding the testing methods, validating their legitimacy, and determining whether the findings pose a real security risk requires both knowledge and precision.

In this particular case, the report claimed a **0.CL Request Smuggling** issue had been identified. However, inconsistencies in the report stood out immediately. Part of the confusion was my own: I initially thought the issue was **CL.0 Request Smuggling**, a vulnerability I was very familiar with. However, once I realized my mistake, I came to the realization that I was only vaguely familiar with **0.CL Request Smuggling**.

After some research and conversations with others in the community, I want to clarify my understanding—and potentially help others—I'll break down the differences between Request Smuggling types and the specific discrepancies between **CL.0 vs. 0.CL**.

---------------------------------------------------------------------------------------------------
## What is Request Smuggling
Request Smuggling is a vulnerability that exploits inconsistencies in how a server or servers handle HTTP requests. These attacks manipulate the communication between servers, and sometimes clients, potentially allowing attackers to hijack sessions, expose sensitive data, or execute unauthorized actions.

### Server-Side Request Smuggling
Server-Side Request Smuggling occurs when differences in request parsing between two backend systems (such as a proxy and an application server) are exploited. For example:

- **Targets**: API gateways, load balancers, and reverse proxies.
- **Delivery Methods**: Sending maliciously crafted HTTP headers (Content-Length vs. Transfer-Encoding).
- **Goal**: Hijack connections, perform cache poisoning, or bypass authentication.
For more details, refer to PortSwigger's [HTTP Request Smuggling](https://portswigger.net/web-security/request-smuggling) documentation.

### Client-Side Request Smuggling / Client-Side Desync
Client-side request Smuggling, also known as desync attacks, focuses on discrepancies in browser-to-server communication. These attacks exploit misconfigured/misguided request handling between the browser and application server to enable same-origin bypasses, credential theft, or cross-user attacks.

For an overview, James Kettle's research on [browser-powered desync attacks](https://portswigger.net/research/browser-powered-desync-attacks) is an excellent resource.

---------------------------------------------------------------------------------------------------

## What is CL.0 Request Smuggling
CL.0 is a server-side vulnerability caused by discrepancies between how two backend servers interpret an HTTP request. The vulnerability exists between backend servers (Application servers behind a proxy/load balancer). The server is simply not expecting a body or is not set up to handle a body. This results in the server leaving the body on the wire, waiting for the next request to be appended to it. There is not a lot of reason for this to happen.  I've identified this issue before and the source ended up being an SSL server incorrectly handling the `Connection: keep-alive` header.

The **victim/target** in this case would be other users on the platform or restricted applications behind a firewall. This is because other requests can be manipulated or you can manipulate your own request if you're fast enough.

> Request Smuggling is an indeterminate attack. You cannot guarantee that you will affect your own subsequent request or a specific user's request.
{: .prompt-tip}

### Testing Methodology
CL.0, meaning **Content-Length.0**, is one of the easier Request Smuggling vulnerabilities to test for. In this scenario, the front-end server uses the `Content-Length`, but the back-end, for some reason, ignores it completely. As a result, the back-end treats the body as the start of a second request, ignoring the `Content-Length`. This is equivalent to the back-end treating the first request as having a `Content-Length` of 0. There are a couple of ways to test for this, but here's one:

#### 1. Find an endpoint that supports `GET` and `POST` requests. 
One way you can test for this is by selecting a `GET` request, sending it to Repeater, right-clicking the request and selecting "Change request method". If you send the new request as a `POST` request and the application responds normally, then move on to the next step. This indicate that the applciation may not be processing the body.

> Try to look for endpoints that might not be expecting a body, like files and images. I found this issue on an endpoint that indicated that there was a proxy. Something like `/proxychache/assets/main.js`.
{: .prompt-tip}

{:start="2"}
#### 2. Create a request in Burp Suite's Repeater 
This request should have these key attributes:

| Attribute                             | Description                                                                                                                                                                        |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `POST` request                        | The request method used to test for the vulnerability.                                                                                                                             |
| `Connection: keep-alive`              | The connection needs to remain open if we want to affect our own or others' requests.                                                                                              |
| `Content-Length: 23`                  | The `Content-Length` needs to be the actual, correct content length. The vulnerability occurs because it's not expecting a body.                                                   |
| Smuggled request for a valid endpoint | The smuggled request needs to be for an endpoint we know exists. We do this so when we send a request that we know will be a 404 and it returns to 200, the attack was successful. |

and should look something like this <i class="arrow-circle-down"></i>

```http
POST /vulnerable-endpoint HTTP/1.1
Host: vulnerable-webiste.com
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 23

GET / HTTP/1.1
Foo: bar
```

{:start="3"}
3. Add a second request to repeater which is known to return a `404` not found response.

```http
GET /404.html HTTP/1.1
Host: vulnerable-webiste.com

```

{:start="4"}
4. Add both requests to a group in the order we created them.
5. Configure repeater to **Send group in sequence (single connection)**.
6. Send the request and observe the response.

This is just a testing methodology, but it's a good start and covers the basic concepts of CL.0. There are more methods to test for this in the wild. For example, the request smuggling issue I identify this ^^ didn't work. I had to send a large amount of requests quickly in order to observe the smuggled request's response. My guess is this is due to the vulnerability being present several servers down the chain from the server I was requesting from and other users on the platform.

## What is 0.CL Request Smuggling
0.CL, meaning **0.Content-Length**, is a client-side vulnerability stemming from discrepancies between the browser (client) and the application server's request handling. In this scenario, the front-end server ignores the `Content-Length`. This reduces the attack surface to the client sending the request.

### Testing Methodology
todo:

> When testing for client-side request smuggling issues, be sure that you're using requests that a browser can actually send. This means don't put a different content-type than what is actually there, don't change the content-length to something that is not correct, and generally make the request a valid request. The vulnerability should be there without having to misconfigure an HTTP RFC-compliant request.
{: .prompt-warning}

> The application <u>cannot</u> support HTTP/2, otherwise, the desync would not occur as the browser defaults to the highest protocal supported.
{: .prompt-danger}

---------------------------------------------------------------------------------------------------

## So what happened to the issue?
Now that we've got a good background of the issue being reviewed, this was the testing methodology, i.e., the steps required to reproduce the issue under review.

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

4. Group the two requests into a single sequence in Burp Suite's Repeater.
5. Configure the settings to send the group in sequence.
6. Hit Send and review the response for the second request

### Methodology Analysis
The testing methodology relied on manual manipulation of the `Content-Length` header, which can lead to unintended behavior in Burp Suite. This introduces potential false positives if not carefully managed.

Ultimately, the issue was invalid. The tester misinterpreted Burp Suite's behavior when manually setting the `Content-Length` header, mistakenly identifying a 0.CL vulnerability.


--------------------------------------------------------------------------------------------------------------
## Takeaways
While CL.0 and 0.CL shares some underlying principles, but their attack vectors and potential impacts are distinct:

- **CL.0**: Server-to-server discrepancies leading to backend exploitation.
- **0.CL**: Browser-to-server discrepancies enabling client-side exploitation.
Understanding these differences is crucial for accurate issue verification and mitigation.

> Additional Resources:
> - [PortSwigger's Academy Module on CL.0](https://portswigger.net/web-security/request-smuggling/browser/cl-0)
> - [James Kettle's Research Paper](https://portswigger.net/research/browser-powered-desync-attacks)
>
{: .prompt-tip}
