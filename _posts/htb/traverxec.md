---
layout: single_c
title:  "Hack The Box - Traverxec Writeup"
date:   2019-10-17 10:43:16 +0530
categories: OSCP
tags: HTB
classes: wide
---
### Hack The Box - Traverxec

## Enumeration
Lets start by enumerating

#### Nmap
```bash
root@kali:~# nmap -sC -sV 10.10.10.165
Starting Nmap 7.80 ( https://nmap.org ) at 2019-12-02 18:36 IST
Nmap scan report for 10.10.10.165
Host is up (0.24s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.54 seconds
```

There is a `nostromo` server running on port 80

#### Searchsploit
I quick search using `searchsploit` returned some interesting vulnerabilities.

