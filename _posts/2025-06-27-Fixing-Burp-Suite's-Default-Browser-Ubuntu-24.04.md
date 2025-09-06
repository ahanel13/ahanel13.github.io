--- 
title: How to Securely Fix Burp Suite's Default Browser on Ubuntu 24.04
date: 2025-06-27 17:00:00 -0600 # 5:00 PM
author: anthony   
categories: [Miscellaneous, Troubleshooting]
tags: [troubleshooting] # TAG names should always be lowercase
media_subpath: /assets/posts/fixing-burp-suite-browser-ubuntu-24.04
image:
  path: Ubunut-Chromium-logo.png
  alt: Merged Ubunut and Chromium-logo
description: Burp Suite's built-in browser not launching on Ubuntu 24.04? This post will go over a more secure way to fix it with AppArmor, while avoiding dangerous SUID "fixes" that don't follow the Principle of Least Privilege and degrade your systems security.
---

If you're a Web Pentester using Burp Suite on Ubuntu 24.04, you may have encountered a frustrating issue: the built-in Burp browser simply refuses to launch. This problem stems from [a change in how Ubuntu 24.04 handles unprivileged user namespaces](https://ubuntu.com/blog/whats-new-in-security-for-ubuntu-24-04-lts#Unprivileged%20user%20namespace%20restrictions:~:text=22.04%20LTS.-,Unprivileged%20user%20namespace%20restrictions,-Unprivileged%20user%20namespaces), a security feature crucial for the browser's sandboxing. Let's delve into the issue and, more importantly, explore the *right* way to fix it, while steering clear of a dangerously wrong approach that's been circulating online.

# The Root of the Problem: Ubuntu's Security Enhancements

Ubuntu 24.04, in its quest for enhanced security, has tightened restrictions on unprivileged user namespaces. These namespaces are a core Linux kernel feature that allows users to create isolated, sandboxed environments. Burp Suite's built-in browser relies on this sandboxing to isolate web content and protect your system. However, Ubuntu's new, stricter AppArmor configuration now blocks the creation of certain capabilities within these namespaces by default. AppArmor, a powerful security module in Ubuntu, controls what programs can and cannot do. Because Burp's browser needs these capabilities to create its sandbox, it fails to launch.

# The Correct Approach: AppArmor to the Rescue

The recommended and *safe* solution involves creating a custom AppArm   or profile for Burp's browser. This profile explicitly grants the browser the necessary permissions to create its sandbox without compromising system security. Here's the profile you need:

> Best practice would be to replace `@{HOME}/BurpSuitePro/burpbrowser/*/chrome` with the actual path to your Burp browser's. This is typically found in the Burp Suite installation directory. You can use the `find` command to locate it:
> ```bash
> find ~ -type f -name "chrome"
> ```
{: .prompt-warning}

```bash
# This profile allows everything and only exists to give the
# application a name instead of having the label "unconfined"
abi <abi/4.0>,
include <tunables/global>

profile burp-browser @{HOME}/BurpSuitePro/burpbrowser/*/chrome flags=(unconfined) {
  # This grants permission for the application to use unprivileged user namespaces.
  userns,

  # Site-specific additions and overrides. See local/README for details.
  include if exists <local/burpbrowser>
}
```
Save this ^^ content to a file named `/etc/apparmor.d/burpbrowser` (you'll need `sudo` privileges). Then, load the profile using the following command:

```bash
sudo apparmor_parser -r /etc/apparmor.d/burpbrowser
```

> The key part of this solution is the `userns,` line within the profile. This explicitly grants the Burp browser the `userns` (user namespace) permission, allowing it to function correctly.
{: .prompt-tip}

# Avoid the Common SUID "Fix"

A common solution found on forums like [Reddit](https://www.reddit.com/r/bugbounty/comments/1db1lh5/the_default_browser_in_burp_suite_isnt_launching/) involves using `chown` and `chmod` to modify the `chrome-sandbox` binary. This approach, while seemingly simple, could be dangerous and should probably be avoided. This fix typically looks like this:

1.  **Find the `chrome-sandbox` binary:**

    ```bash
    find ~ -type f -name "chrome-sandbox"
    ```

2.  **Change ownership and permissions:**

    ```bash
    sudo chown root:root /path/to/chrome-sandbox && sudo chmod 4755 /path/to/chrome-sandbox
    ```

> The critical part is the `chmod 4755`. The `4` sets the SUID (Set User ID) bit. When the SUID bit is set, *any* user running the `chrome-sandbox` binary will execute it with *root* privileges.
{: .prompt-danger}

**Why is this so bad?** While the sandbox process is designed to drop its elevated privileges after creating the sandbox, running it with SUID as root is not the primary recommendations as it introduces some unnecessary security risks:

  * **Massive Security Risk:** You're giving the browser's sandbox component the ability to run as *root*, the all-powerful user. If a malicious website or attacker finds *any* vulnerability in the browser or if the sandbox wasn't created correctly such as failing to drop privileges, they could potentially escape the sandbox and gain complete control of your system. The would be no need to privilege escalation exploits, as the browser is already running with root privileges.
  * **Ignores Ubunut Security:** Whether you agree or not, Ubuntu has decided to move in the direction of unprivileged user namespaces. They want to enhance security by preventing processes from running with unnecessary privileges. While the chrome-sandbox binary is designed to run with elevated privileges which it drops after creating the sandbox, forcing it to run as root defeats the purpose of this security model.
  * **Unnecessary Privilege Escalation:** The browser doesn't *need* root privileges to create a sandbox. The AppArmor solution demonstrates that.

# Safety First - Principle of Least Privilege
The Principle of Least Privilege (PoLP) is a fundamental security principle that states that users and programs should operate using the least amount of privilege necessary to perform their tasks. By applying this principle, you minimize the potential damage that can occur if a program is compromised.

While the SUID "fix" a _is_ quick and easy way to get Burp's browser working, it introduces an unnecessary privilege escalation path. The AppArmor solution is slightly more involved, but it's Ubuntu's _and_ Portswigger's *recommended* way to fix the problem without compromising your system's security. Always prioritize security best practices and avoid solutions that involve granting unnecessary and excessive privileges.


> The apparmour method can be set and forget as well. The SUID method requires you to remember to change the permissions every time you update Burp Suite, which can be a hassle. The AppArmor profile will remain in place until you decide to remove it, making it a more convenient and secure solution.
{: .prompt-tip}

