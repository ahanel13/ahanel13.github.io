---
title: "How I Redirected Every App User with Request Smuggling"
date: 2026-10-19 09:00:00 -0600
author: anthony
categories: [Vulnerability Analysis, Request Smuggling]
tags: [request smuggling, host header injection, session hijacking, security testing, burp suite, how-to]    # TAG names should always be lowercase
media_subpath: /assets/posts/how-i-redirected-every-app-user
image:
  path: banner.png
  alt: "Diagram of a front-end and back-end disagreeing on an HTTP request boundary, leaving a smuggled remainder between them"
description: How chaining a low-severity host header reflection with an HTTP request smuggling bug let me redirect every user on an application and capture admin sessions.
---


> This is a sanitized account of a finding from an authorized penetration test. All client-identifying details, endpoints, and payloads have been changed or generalized.
{: .prompt-info}

Two of the most fun findings I've ever written up came from the *same* underlying bug on the *same* engagement. On their own, the ingredients weren't especially exciting: a **request smuggling** vulnerability, and a **low-severity host header reflection** on a navigation endpoint. Chained together, though, they let me redirect *every* user on the application to any domain I wanted, and, separately, capture the session of *any* user, including admins.

This is the story of how a "low" and a "medium" combined into two outcomes I'd call critical, and how the finished report still went out calling the whole thing a medium.

---------------------------------------------------------------------------------------------------
## The Two Ingredients

### 1. A Request Smuggling Vulnerability
The application was fronted by a proxy that disagreed with the back-end about where one HTTP request ended and the next began. That's the heart of any [request smuggling](https://portswigger.net/web-security/request-smuggling) issue: the front-end and back-end parse request boundaries differently, so I could sneak ("smuggle") a partial request that would get prepended to whatever request came down the same connection next.

In this case the disagreement lived in a back-end **Oracle HTTP Server** (OHS) instance. OHS was adding TLS to the traffic passing through it, and it misread where the HTTP requests inside that traffic started and stopped. The front-end and OHS were reading the same bytes off the same socket and splitting them into requests in two different places, which is all a desync ever really is.

> If you want a refresher on the different smuggling variants and how to test for them, I wrote about that in [Understanding CL.0 and 0.CL Attacks](/posts/Understanding-0.CL-and-O.CL-Attacks-A-Comparative-Guide/).
{: .prompt-tip}

Two properties of this bug mattered for what follows:

- I could **poison a shared connection** so that a *victim's* request was affected, not just my own.
- I controlled a **`Content-Length`** that the back-end would honor, letting me decide how many bytes belonged to my smuggled request versus the victim's.

### 2. A Host Header Reflection on a Navigation Endpoint
Separately, there was a navigation endpoint that reflected the incoming [`Host` header](https://portswigger.net/web-security/host-header) straight back into its response, specifically into a redirect. Send it an arbitrary `Host`, and the server would happily issue a `302` pointing at whatever host you gave it.

```http
GET /navigate?to=dashboard HTTP/1.1
Host: attacker-controlled.example

HTTP/1.1 302 Found
Location: https://attacker-controlled.example/dashboard
```

On its own, this is a textbook low. To exploit it you'd have to convince a victim to send a request with an attacker-controlled `Host` header, and browsers don't let you do that. So it sat there as an unexciting reflection issue... until I had a way to control other users' requests.

---------------------------------------------------------------------------------------------------
## Attack #1: Redirecting Every User on the Application

The host header reflection needed a request with a malicious `Host` to reach the vulnerable endpoint. The request smuggling vulnerability was *exactly* the primitive that could deliver one into someone else's connection. PortSwigger catalogues this precise pairing as [turning an on-site redirect into an open redirect](https://portswigger.net/web-security/request-smuggling/exploiting#using-http-request-smuggling-to-turn-an-on-site-redirect-into-an-open-redirect).

The idea:

1. Smuggle a request to the navigation endpoint carrying an **attacker-controlled `Host` header**.
2. Because of the smuggling bug, that smuggled prefix gets stitched onto the front of the **next user's request** on the poisoned connection.
3. The back-end responds to *their* request as if it were mine, reflecting my malicious `Host` into a `302 Location`.
4. The victim's browser follows the redirect to my domain.

```http
POST / HTTP/1.1
Host: vulnerable-website.example
Content-Length: 87
Transfer-Encoding: chunked

0

GET /navigate?to=dashboard HTTP/1.1
Host: attacker-controlled.example
X-Ignore: 
```
*Illustrative shape of the smuggled prefix, in the classic [CL.TE](https://portswigger.net/web-security/request-smuggling#cl-te-vulnerabilities) form. The front-end honors `Content-Length` and forwards all 87 bytes; the back-end stops at the `0` chunk, leaving everything after it buffered as the head of the next request. The dangling `X-Ignore:` swallows the victim's real request line so what follows still parses.*

Because the poisoned request sits at the head of the connection's queue, *whoever* came through next got redirected. By repeatedly re-poisoning the connection(s), I could keep the queue filled and effectively redirect **every user** on the application to any endpoint or domain I chose.

> Request smuggling is **indeterminate**: you can't guarantee *which* user you'll hit on any given request. But you don't have to. Keep filling the queue and, over enough requests, everyone gets caught.
{: .prompt-warning}

The impact writes itself: a mass redirect to an attacker-controlled domain is a ready-made phishing and credential-harvesting delivery mechanism, served from the *legitimate* application's own traffic. No email, no lure: just visit the real site and get sent somewhere else.

---------------------------------------------------------------------------------------------------
## Attack #2: Capturing Any User's Session

The second attack used the same smuggling bug but a different primitive: an **over-large `Content-Length`**. The Academy covers this one as [capturing other users' requests](https://portswigger.net/web-security/request-smuggling/exploiting#capturing-other-users-requests).

The account I was testing with had access to a **comment feature**, an ordinary "leave a comment" box that stored and displayed whatever text I submitted. That turned out to be the perfect place to catch other people's requests.

Here's the trick. I smuggled a request that:

- Targeted the comment-submission endpoint, and
- Declared a `Content-Length` **larger than the body I actually sent**.

The back-end, waiting to receive the full number of bytes I promised, treated the **victim's incoming request as the rest of my comment body**. Their request line, their headers, including their session cookie, got appended to my comment and stored.

```http
POST /comments HTTP/1.1
Host: vulnerable-website.example
Content-Type: application/x-www-form-urlencoded
Content-Length: 700

comment=
```
*The body is far shorter than the declared `Content-Length`, so the server keeps reading, straight into the next user's request.*

All I had to do then was **read my own comment**, where the victim's captured request was now sitting in plain text:

```http
comment=GET /account HTTP/1.1
Host: vulnerable-website.example
Cookie: session=<victim-session-token>
...
```

With a captured session token, I could ride that user's authenticated session. And because the attack is indeterminate about *who* it catches, that "any user" repeatedly included **admin** users: full account takeover of the highest-privileged accounts on the platform.

---------------------------------------------------------------------------------------------------
## What Else It Could Have Delivered
The two attacks I wrote up are the two I built. They weren't the ceiling, they were where the time box ran out. Both primitives generalise: I controlled what sat at the head of a victim's connection, and I controlled how many bytes the back-end kept reading. From there the list gets long.

- **Smuggled JavaScript.** The same prefix that carried a `Host` header could carry a request to any endpoint that reflects input, [delivering script into the victim's response](https://portswigger.net/web-security/request-smuggling/exploiting#using-http-request-smuggling-to-exploit-reflected-xss). Riding in on someone else's connection removes the usual delivery problem: no link to click, no payload to squeeze past a filter in a URL, and the victim's cookies are attached because the request really is theirs.
- **Credential harvesting on the legitimate origin.** If I can control what the victim's browser renders, I can serve a login form whose action points somewhere I control. The victim is on the correct hostname, with a valid certificate and the real URL in the address bar. There is no lookalike domain to notice and nothing that security awareness training teaches people to spot.
- **Plaintext credentials, not just session tokens.** Attack #2 captured whatever happened to be in the victim's request. A session cookie is what I caught. A login POST would have handed over a username and password in the clear, with no session lifetime to limit the damage.
- **[Bypassing front-end security controls](https://portswigger.net/web-security/request-smuggling/exploiting#using-http-request-smuggling-to-bypass-front-end-security-controls).** Paths the proxy refuses to route, admin interfaces and internal endpoints among them, become reachable when the back-end is the thing parsing your smuggled request line.
- **[Web cache poisoning](https://portswigger.net/web-security/request-smuggling/exploiting#using-http-request-smuggling-to-perform-web-cache-poisoning) and [cache deception](https://portswigger.net/web-security/request-smuggling/exploiting#using-http-request-smuggling-to-perform-web-cache-deception).** With a cache in the path, one poisoned response stops being a one-shot. It gets stored and handed to everyone who asks for that resource, and it persists without re-poisoning anything.
- **[Response queue poisoning](https://portswigger.net/web-security/request-smuggling/advanced/response-queue-poisoning).** The end state for this class: desynchronise the response queue itself so users are served each other's responses wholesale, turning an intermittent capture into a steady leak.

None of this is exotic. It is the documented escalation path for the vulnerability class, and most of it is a lab you can work through in an afternoon. That is what made the rating conversation frustrating: the two attacks in the report were the floor, not the ceiling.

---------------------------------------------------------------------------------------------------
## Why This Chain Was So Dangerous
Individually, the pieces were rated modestly. Together they produced two outcomes I'd put squarely in critical territory:

| Chain | Primitive used | Impact |
| ----- | -------------- | ------ |
| Mass redirect | Smuggled `Host` header + host header reflection | Redirect **every** user to an attacker domain (phishing / credential theft at scale) |
| Session capture | Smuggled over-large `Content-Length` + stored comment | Capture **any** user's session, including admins → account takeover |

This is the whole argument for chaining. A host header reflection that "just" echoes a value, and a smuggling bug that's hard to steer, look tolerable in isolation. Wire the deliverable (the reflection) to the delivery mechanism (the smuggling), and you get an attacker who controls where everyone lands and who owns everyone's session.

---------------------------------------------------------------------------------------------------
## The Rating I Didn't Get
I argued for a higher rating. I lost. The issue went out rated **medium**, "due to difficulty of exploitation."

I understand the reasoning, and I want to represent it fairly. Request smuggling is finicky. It leans on connection reuse, on a race you don't control, and on a victim's request landing in the exact window you poisoned. Nobody can promise that a particular admin's session will fall into their hands at a particular moment. If you measure risk as "how likely is one clean, targeted exploit," the difficulty is real, and it should pull the number down.

My argument was that this measures the wrong thing. The attack never needed to be reliable. It needed to be repeatable, and it was:

- **Indeterminacy stops mattering at scale.** Not being able to choose *which* user you hit is irrelevant once you can keep re-poisoning the connection until you've hit all of them. "Unreliable per attempt" and "inevitable over an afternoon" are very different risks wearing the same word.
- **The difficulty had already been paid.** None of this was theoretical. Captured sessions, admin ones included, were sitting in my own comment thread. Whatever the exploit cost in effort, that cost was sunk, inside a time-boxed test, by one person who wasn't even trying to be subtle about it.
- **Difficulty isn't a control.** It's a measure of attacker patience, and it decays. Techniques get published and tooling gets better, while the application's exposure doesn't improve just because exploitation is awkward this year.
- **I handed over a working script.** This is the part that still bothers me. The objection was that exploitation was extremely difficult, and the finding shipped with automation that performed it. Whatever difficulty existed was research cost, and research cost gets paid once, by whoever goes first.

I'll be careful here, because I'm guessing at other people's reasoning. But I don't think the disagreement was really about likelihood. I think it was about familiarity. Request smuggling sits in an awkward spot: thoroughly documented, extensively labbed, and still genuinely unfamiliar to teams who have never had to think about two servers disagreeing over a byte count. The mechanics are counterintuitive the first time you meet them. When a bug class is unfamiliar, "difficult" and "unfamiliar" start getting used as though they were the same word, and only one of them belongs in a risk rating.

"Difficulty of exploitation" is a fair discount on the attacks you *described*, and a poor one on the attacks you *performed*. If a finding contains someone else's session token and a script that goes and fetches more, the hard part is already behind you.

So yes, I think this one was mislabeled. Not through negligence and not in bad faith, but because the rating ended up reflecting how hard the issue was to *understand* rather than how hard it was to *exploit*. Those two feel like the same thing when you're meeting a vulnerability class for the first time. They are not.

And the critique I'd level at myself is simpler than any of that: a good share of the understanding gap I was frustrated by was mine to close. I understand this attack class far better now than I did while I was arguing about it. At the time I could describe what I had done, but I hadn't worked out how to make someone else *feel* the reach of it, which is a different skill. Handing over a script proves the exploit runs. It doesn't transfer the understanding, and understanding is what a rating conversation actually runs on.

What I'd do differently is lead with the escalation ladder rather than the two attacks I happened to finish, because the ladder is the argument. The captured cookie is interesting; the short distance from that cookie to a credential harvesting form on the client's own domain is the part that changes how someone votes. I'd also walk the demonstration through live with the people weighing the risk, rather than leaving a script in an appendix for them to run on their own. If a finding's severity depends on the reader grasping an unfamiliar mechanism, making that mechanism legible isn't something I do as a favour after the report lands. It's part of the finding.

---------------------------------------------------------------------------------------------------
## Remediation
- **Fix the request smuggling at the source.** Ensure the front-end and back-end agree on request boundaries: normalize `Content-Length`/`Transfer-Encoding` handling, reject ambiguous requests, and prefer back-end connections that don't reuse a single socket across users. This is the primitive that made *both* attacks possible; kill it and the chain collapses.
- **Stop reflecting the `Host` header.** Validate the incoming `Host` against an allow-list of known-good hostnames and build redirects from server-side configuration, never from client-supplied headers.
- **Don't build redirects from untrusted input.** Treat the navigation endpoint's `Location` as security-sensitive.
- **Patch the component doing the misreading.** The desync originated in how OHS parsed requests while adding TLS, so the durable fix lives there rather than in workarounds layered on at the proxy. Keep OHS current with Oracle's Critical Patch Updates, and re-test the boundary behaviour afterward instead of assuming a version bump settled it.

---------------------------------------------------------------------------------------------------
## Further Reading
PortSwigger's Web Security Academy covers all of this in more depth than one writeup can, and its labs let you build the attacks yourself rather than read about them:

- [HTTP request smuggling](https://portswigger.net/web-security/request-smuggling), the vulnerability class itself, including the [CL.TE](https://portswigger.net/web-security/request-smuggling#cl-te-vulnerabilities) and [TE.CL](https://portswigger.net/web-security/request-smuggling#te-cl-vulnerabilities) variants
- [Finding request smuggling vulnerabilities](https://portswigger.net/web-security/request-smuggling/finding), where the timing techniques live
- [Exploiting request smuggling](https://portswigger.net/web-security/request-smuggling/exploiting), which covers both techniques I used here: [turning an on-site redirect into an open redirect](https://portswigger.net/web-security/request-smuggling/exploiting#using-http-request-smuggling-to-turn-an-on-site-redirect-into-an-open-redirect) and [capturing other users' requests](https://portswigger.net/web-security/request-smuggling/exploiting#capturing-other-users-requests)
- [HTTP Host header attacks](https://portswigger.net/web-security/host-header), and [exploiting them](https://portswigger.net/web-security/host-header/exploiting)
- [Browser-powered desync attacks](https://portswigger.net/web-security/request-smuggling/browser), for the modern CL.0 and client-side variants

---------------------------------------------------------------------------------------------------
## Takeaways
1. **Rate findings for their chains, not just their isolated severity.** A "low" that becomes the payload for a "medium" can be a critical, whatever the finished report ends up calling it.
2. **Request smuggling is a *delivery* primitive.** The interesting question is rarely "can I desync?"; it's "what can I now deliver into someone else's connection?"
3. **Any place that stores and shows your input is a candidate exfiltration sink.** A humble comment box became the readout for captured victim requests.
4. **A rating is only as sound as the shared understanding behind it.** If the people weighing the risk haven't internalised the attack, "difficult" quietly becomes a stand-in for "unfamiliar." Closing that gap is the tester's job as much as anyone's, and it's the part of this engagement I'd do differently.

If you take one thing from this: when you find a smuggling bug, don't stop at the proof-of-concept `404`→`200`. Go look for the low-severity reflections and the stored-input features already sitting in your notes. That's where the criticals are hiding.
