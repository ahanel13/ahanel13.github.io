---
title: wsrepl GitHub Project Contribution
date: 2023-09-30 12:00:00 /-0600
author: anthony   
categories: [Projects, Open-Source Contributions]
tags: [contributions, dev work, open-source]     # TAG names should always be lowercase
media_subpath: /assets/posts/wsrepl
description: I contributed to a WebSocket Pentesting tool that was sponsored by Doyensec on GitHub.
---

## Background
I initially saw James Kettle’s post about this new [WebSocket REPL project](https://github.com/doyensec/wsrepl). I happened to be testing a chatbot that was using Websockets to communicate at the time. The current version of Burp Suite is a little clunky when it comes to testing WebSockets. Especially in this scenario, since to create a chat there were multiple HTTP requests needed before a chat ID would be given and a WebSocket could even be opened. 

While I really liked what [Andrew Konstantinov](https://github.com/execveat) had put together for testing, it just wasn’t robust enough to handle the chat creation flow I was testing at the time. After looking over the code, I felt confident that I could add this feature and make it work at least for my use case. 

## James Kettle's LinkedIn Shoutout
<div style="text-align: center;">
  <iframe allowfullscreen="" src="https://www.linkedin.com/embed/feed/update/urn:li:share:7087097423633817602?wmode=opaque" width="90%" data-embed="true" frameborder="0" title="Embedded post" height="550"></iframe>
  <p>
    <a href="https://www.linkedin.com/feed/update/urn:li:share:7087097423633817602">Link to article</a>.
  </p>
</div>


## WebSocket REPL for pentesters
-------------------------------
![Desktop View](Doyensec_Logo_2019_10_15-02_bisGray.png){: width="972" height="589" .w-50 .right}
**wsrepl** is an interactive WebSocket REPL designed specifically for penetration testing. It provides an interface for observing incoming WebSocket messages and sending new ones, with an easy-to-use framework for automating this communication.

## Pull Request: Enable Dynamic WSS Endpoints via Plugins
--------------------------------------------------------
Since my test case at the time required three requests before a WSS link was provided, and the link would have a dynamic variable indicating the session I need to add a method of how the code accepted WSS links. 

In the project’s original form, I would have to copy over the new WSS every time, making it just as clunky as Burp Suite. I also couldn't write a plugin for Auth since the WSS link was dynamic. 

However, updated the code and created a [Pull Request](https://github.com/doyensec/wsrepl/pull/3#issue-1826429768) which was accepted. Now the code accepts plugin-provided URLs if specified when executing wsrepl.
