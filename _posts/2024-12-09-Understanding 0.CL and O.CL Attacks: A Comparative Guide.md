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

  h2 {
    color: white;
  }

  h4 {
    font-style: italic;
  }
</style>

I recently reviewed a reported issue regarding 0.CL. While reviewing issues isn't part of my daily responsibilities, it's something I encounter occasionally. The process can be challenging—understanding the testing methods, validating their legitimacy, and determining whether the findings pose a real security risk requires both knowledge and precision.

In this particular case, the researcher claimed to have found a **0.CL Request Smuggling** issue. However, inconsistencies in the report stood out immediately. Part of the confusion was my own: I initially approached the issue as if it was **CL.0 Request Smuggling**, a vulnerability I was very familiar with, having identified and presented on it in the past. However, after a conversation with the researcher, I realized my mistake and was only vaguely familiar with **0.CL Request Smuggling**.

Before I could say anything about the legitimacy of the issue, I had to do some learning. After some research and conversations with others in the community, I acquired a better understanding of the issue. I realized there are not many resources on **0.CL** *specifically*. James Kettle's research [ Browser-Powered Desync Attacks: A New Frontier in HTTP Request Smuggling](https://portswigger.net/research/browser-powered-desync-attacks) is the only instance I was able to find where **CL.0** was mentioned and it was only indirectly mentioned, not *really* talked about.

![Image of table from James Kettle's research paper](0.CL-table-image.png)

 So to help solidify my understanding and potentially help others, I'll attempt to break down the differences between Request Smuggling types and the specific differences *and* similarities between **CL.0 vs. 0.CL**.

---------------------------------------------------------------------------------------------------
## What is Request Smuggling
Request Smuggling is a vulnerability that exploits inconsistencies in how a server or servers handle HTTP requests. These attacks manipulate the communication between servers, and sometimes clients, potentially allowing attackers to hijack sessions, expose sensitive data, or execute unauthorized actions.

### Server-Side Request Smuggling
Server-side request smuggling occurs when two backend systems, such as a proxy and an application server, parse the ends of HTTP requests differently. This discrepancy can be exploited by attackers to sneak a malicious request between the servers that interpret the request boundaries inconsistently. Attackers often target API gateways, load balancers, and reverse proxies by sending specially crafted HTTP requests with ambiguous `Content-Length` and `Transfer-Encoding` headers.  

The vulnerability is most common in `HTTP/1.1` due to its flexible interpretation of headers and chunked transfer encoding, which can lead to parsing discrepancies between upstream and downstream servers. In contrast, `HTTP/2.0` and `HTTP/3.0` use binary protocols with stricter framing and parsing rules, reducing the likelihood of such discrepancies. However, smuggling attacks can still occur in scenarios where protocols are [downgraded](https://portswigger.net/web-security/request-smuggling/advanced/http2-downgrading) to `HTTP/1.1` during transmission or where improper translation between protocols takes place.

> For more in-depth details on different types of request smuggling, see PortSwigger's [HTTP Request Smuggling](https://portswigger.net/web-security/request-smuggling) documentation.
{: .prompt-tip}

### Client-Side Request Smuggling / Client-Side Desync
Client-side request Smuggling, used in client-side desync (CSD) attacks, focuses on discrepancies in browser-to-server communication. These attacks exploit misconfigured/misguided request handling between the browser and application, enabling a new attack surface including CSRF-style attacks, same-origin bypasses, and interestingly *some* server-side request smuggling attacks.

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
One way to test for this is by selecting a `GET` request, sending it to Repeater, right-clicking the request, and selecting "Change request method." If you send the new request as a `POST` request and the application responds normally, then move on to the next step. This indicates that the application may not be processing the body.

> Try to look for endpoints that might not be expecting a body, like files and images. I found this issue on an endpoint that indicated that there was a proxy. Something like `/proxychache/assets/main.js`.
{: .prompt-tip}

#### 2. Create a request in Burp Suite's Repeater
This request should have these key attributes:

| Attribute                             | Description                                                                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `POST` request                        | The request method used to test for the vulnerability.                                                                                                                                     |
| `Connection: keep-alive`              | The connection needs to remain open if we want to affect our own or others' requests.                                                                                                      |
| `Content-Length: 24`                  | The `Content-Length` needs to be the actual, correct content length. The vulnerability occurs because it's not expecting a body.                                                           |
| Smuggled request for a valid endpoint | The smuggled request needs to be for an endpoint we know exists. We do this so that when we send a request that we know will be a `404` and it returns to `200`, the attack is successful. |

and should look something like this  <i class="fa fa-arrow-circle-down"></i>

```http
POST /vulnerable-endpoint HTTP/1.1
Host: vulnerable-webiste.com
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 24

GET / HTTP/1.1
Foo: bar
```

#### 3. Add a second request to Repeater which is known to return a `404` not found response.

```http
GET /404.html HTTP/1.1
Host: vulnerable-webiste.com

```

#### 4. Configure Repeater to Send Requests in Group
- Add both requests to a group in the order we created them.
  ![Group requests option in Repeater](Create-Group-Repeater_Option.png)

- Configure repeater to send group in sequence (single connection).
  ![Sending Repeater group options](Send-Group-Repeater-Option.png)

#### 5. Send the request and observe the response.

### Key Considerations
This testing methodology serves as a solid starting point for understanding the basic concepts of CL.0. However, there are additional methods to explore when testing in real-world scenarios. For instance, the request smuggling issue I identified did not yield results initially. To observe the response to the smuggled request, I needed to send a large volume of requests in quick succession. I suspect the vulnerability may exist on a server further downstream from the one I was targeting. Additionally, the activity of other users on the platform could have reduced the likelihood of detecting changes from my smuggled requests.

## What is 0.CL Request Smuggling / Client-Side Desync
0.CL, meaning **0.Content-Length**, is a client-side vulnerability stemming from discrepancies between the browser (client) and the application server's request handling. In this scenario, the front-end server ignores the `Content-Length`. This reduces the attack surface to the client sending the request.

### Testing Methodology
Testing for 0.CL CSD requires a systematic approach to identify discrepancies in how the front-end server and the back-end server handle requests. We can test for this by sending a request with `Content-Length` being larger than what is actually within the body. If the application hangs, then the `Content-Length` is being processed and the app is waiting for the rest of the body. If the app responds immediately, then this is work investigating further. Here's what that would look like:

#### 1. Ensure correct conditions are met
 - `HTTP/2` is *not* support
 - HTTP Pipelining *is* supported

#### 2. Create Request with larger Content-Length
The request should probably have a body and the `Content-Length` should be set to be larger than the body. Be sure to update Burp Suite's Repeater settings to avoid automatic updating of the `Content-Length`.

```http
POST /vulnerable-endpoint HTTP/1.1
Host: vulnerable-webiste.com
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 4

Foo=bar
```

#### 3. Send this request and observe
When the request is sent, observe whether the application will hang and wait for the rest of the request or if it responds quickly. Is the application response different if you change the type of payload/`Content-Type` or is it the same?

#### 4. Confirm Desync
Once you've identified an endpoint that appears to be ignoring the body and meets the previous requirements, we can test one more thing. Does the application parse the body correctly and just ignore it or is the server actually leaving the body to be processed later? We'll send two requests down the same connection using the method in the [CL.0 testing methodology](#testing-methodology).

> The `Content-Length` *must* be correct when confirming CSD.
{: .prompt-danger}

#### 5. Create Desync POC
Once we've confirmed the a desync is occuring we attempt to recreate the issue on a vicitm by building the attack from the browser.

1. Navigate to a website with HTTPS that is not the website we've been targeting (vulnerable-webiste.com).
2. Open the browser dev tools and configure the **Preserve Log** and **Connection ID** options in the **Network** tab.
![Browser Network Options](Browser-Network-Options.png)
3. Switch to **Console** tab and use JavaScript `fetch()` to recreate the request from [step 4](#4-confirm-desync)

```javascript
fetch('https://vulnerable-website.com/vulnerable-endpoint', {
    method: 'POST',
    body: 'GET /404 HTTP/1.1\r\nFoo: x', // smuggled request prefix
    mode: 'no-cors', // ensures the connection ID is visible on the Network tab
    credentials: 'include' // poisons the "with-cookies" connection pool
}).then(() => {
    location = 'https://vulnerable-website.com/' // uses the poisoned connection
})
```

> The steps above were taken from [Web Security Academy](https://portswigger.net/web-security/request-smuggling/browser/client-side-desync#probing-for-client-side-desync-vectors)

### Key Considerations
When *confirming* a CSD issue, ensure that your requests are ones that a browser can actually send. This means you should not alter the `Content-Type` from what it actually is, and avoid changing the `Content-Length` to an incorrect value. Make sure that the request is HTTP RFC-compliant, the vulnerability should exist without misconfiguration. Otherwise, you cannot create a working POC.

> The application <u>cannot</u> support HTTP/2, otherwise, the desync would not occur as the browser defaults to the highest protocol supported.
{: .prompt-danger}
---------------------------------------------------------------------------------------------------

## So what happened to the issue?
Now that we've got a good background of the issue being reviewed, this was the testing methodology, i.e., the steps to reproduce the issue under review.

1. Add the following HTTP request to a Repeater tab in Burp Suite:
  *Not the incorrect `Content-Length`*
 
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
{:start="2"}
2. Disable Burp Suite's automatic Content-Length updating.
3. Create an additional `GET` request for the home page:

```http
GET / HTTP/1.1
Host: public.facing.host.com
Connection: Keep-Alive

```

{:start="4"}
4. Group the two requests into a single sequence in Burp Suite's Repeater.
5. Configure the settings to send the group in sequence.
6. Hit Send and review the response for the second request

### Methodology Analysis
The testing methodology relied on manual manipulation of the `Content-Length` header, which is how 0.CL can *first* be identified, but if not updated to be correct during confirmation, can lead to false positives.

Ultimately, the issue was invalid. The tester misinterpreted Burp Suite's behavior when manually setting the `Content-Length` header, mistakenly identifying a 0.CL vulnerability. They could not reproduce the behavior without keeping the `Content-Length` as the incorrect value, meaning this is actually HTTP Pipelining. Since they couldn’t replicate the attack with HTTP RFC-compliant requests, and the "vulnerability" was not exploitable. In other words, not an issue.

--------------------------------------------------------------------------------------------------------------
## Takeaways
While CL.0 and 0.CL shares some underlying principles, such as the way they are confirmed. But their potential impacts are distinct:

- **CL.0**: Server-to-server discrepancies leading to backend exploitation.
- **0.CL**: Browser-to-server discrepancies enabling client-side exploitation.
 
Understanding these differences is crucial for accurate issue verification and mitigation.
