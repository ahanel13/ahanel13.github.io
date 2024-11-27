--- 
title: I Created a Firewall Ferret
date: 2024-09-13 11:00:00 -0600
last_modified_at: 2024-11-27 11:00:00 -0600
author: anthony   
categories: [Projects, Burp Suite Extensions]
tags: [dev work, burp suite, open-source]     # TAG names should always be lowercase
description: With Firewall Ferret, security testers can now have greater control and precision when testing WAFs, manually adding junk data to requests and expanding Burp Suite’s active scan checks.
media_subpath: /assets/posts/firewall-ferret
image:
  path: Gemini_Generated_Image_8722iv8722iv8722.jpg
  alt: Gemini Generated Firewall Ferret
---

Web Application Firewalls (WAFs) are essential for securing web applications from common attacks like SQL injection and cross-site scripting. However, a known limitation is that WAFs often inspect only a limited amount of data per request, leaving them vulnerable to payload padding. This is where my latest Burp Suite extension, **[Firewall Ferret](https://github.com/ahanel13/Firewall-Ferret)**, comes into play.

## Why I Built Firewall Ferret
-----------------------------

Firewall Ferret is my solution to address the limitations I noticed in existing WAF bypass tools, such as **[WAF Bypadd](https://github.com/portswigger/waf-bypadd)**, which was built on a legacy API. I had trouble using the older extension, I needed more flexibility and control. Since I have exerience working with Portswigger’s newer **[Montoya API](https://portswigger.github.io/burp-extensions-montoya-api/javadoc/burp/api/montoya/MontoyaApi.html)** I decided that recreating and extending the extension would be the best use of my time. Single page python projects are not fun to work on.

With Firewall Ferret, security testers can now have greater control and precision when testing WAFs, manually adding junk data / WAF Bullets to requests and expanding Burp Suite’s active scan checks.

## Key Features of Firewall Ferret
----------------------------------

### 1. Automatic Junk Data Insertion
Firewall Ferret can automatically insert junk data into specific content types, including URL-Encoded, JSON, XML, and Multipart bodies. By padding the payload, the tester can push beyond the typical WAF inspection limit and uncover hidden vulnerabilities.

### 2. Manual Junk Data Insertion
For more control, testers can also manually insert junk data at any point within a request. This feature is handy when you need to target a specific parameter or data point to slip past a WAF.

### 3. Enhanced Active Scans
Firewall Ferret significantly enhances Burp Suite’s default active scans by duplicating each scan check and adding various payload sizes—ranging from 8 KB to 1024 KB—to the beginning of every payload. This increases the chance of evading WAF rules and discovering hidden vulnerabilities.

## Why You May Not Find It in the BAPP Store
--------------------------------------------
![Burp Suite BAPP Store](Burp_Suite_BAPP_Store.png){: .w-50 .right}

Although Firewall Ferret was designed to be a powerful tool for testers, it may not find its way into Portswigger’s BAPP store anytime soon. According to Portswigger's guidance, they do not plan to remove or replace any existing extensions in the BAPP store. Instead, they encourage developers to contribute to or improve existing projects rather than submit new ones.

### Update: Firewall Ferret Approved by Portswigger
--------------------------------------------------

Great news! Portswigger has approved Firewall Ferret, and it is now available in the BAPP store. You can find it here: [Firewall Ferret on BAPP Store](https://portswigger.net/bappstore/ca894f9bab6446f0aa7eac712a7b80ca) or within Burp Suite in the BApp store.

## How to Get Started
---------------------

You can easily install Firewall Ferret by following these steps:

1. **Download the Latest Release**  
  Head over to the project’s [GitHub page](https://github.com/ahanel13/Firewall-Ferret) to download the latest release.
2. **Install in Burp Suite**  
  Add the extension manually via Burp Suite’s Extensions tab, selecting **Firewall Ferret** as a Java extension.

## A Word on WAFs
-----------------

WAFs vary widely in their configuration, and many can be tuned to inspect larger amounts of data. Here's a brief summary of some common WAF limitations:

| **WAF Provider** | **Max Request Body Inspection** |
| ---------------- | ------------------------------- |
| Cloudflare       | 128 KB – 500 MB                 |
| AWS WAF          | 8 KB – 64 KB                    |
| Azure WAF        | 128 KB – 4 GB                   |
| Akamai           | 1 KB – 32 KB                    |

For more information on WAF inspection limits and how Firewall Ferret can help you test these, refer to the full table in the [project README](https://github.com/ahanel13/Firewall-Ferret).

## Conclusion
-------------

Firewall Ferret offers a much-needed upgrade for testers working with WAF bypass techniques, providing an easy-to-use yet powerful tool for evading standard WAF checks. Even though it may not be in the BAPP store, you can still benefit from its functionality by manually installing it from GitHub.

For more details or to download the extension, visit the [Firewall Ferret GitHub Repository](https://github.com/ahanel13/Firewall-Ferret).
