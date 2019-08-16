---
layout: single_c
title:  "Pwnable.tw Level 3 - Calc"
date:   2019-08-16 1:01:16 +0530
categories: Pwnable.tw
tags: ctf
classes: wide
--- 
### [Pwnable.tw Level 3 - Calc](https://pwnable.tw/challenge/#3)  

## Challenge
```
Have you ever use Microsoft calculator?

nc chall.pwnable.tw 10100
```
## Static Analysis
lets check file properties
```bash
itpc@iTPC-LT31:~/Downloads$ file calc 
calc: ELF 32-bit LSB executable, Intel 80386, version 1 (GNU/Linux), statically linked, for GNU/Linux 2.6.24,   
BuildID[sha1]=26cd6e85abb708b115d4526bcce2ea6db8a80c64, not stripped

itpc@iTPC-LT31:~/Downloads$ ./checksec.sh --file calc 
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
Partial RELRO   Canary found      NX enabled    No PIE          No RPATH   No RUNPATH   calc
```
So we have stack `canary` and the `stack` is not executable.