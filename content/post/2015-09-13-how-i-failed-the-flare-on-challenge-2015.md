---
title: "How I Failed the Flare-on Challenge 2015"
date: 2015-09-13
categories:
  - Security Challenge
tags:
  - Reverse Binary
draft: false
toc: true;
---

The second Flare-on Challenge was ended last Tuesday. I am only on the half way when the final is due. I would like to note down some of the pitfall that I realized after the game to remind me in future journey to the final list of similar games. I would elaborate through each challenges and outline the lessons I learned in each.

# Challenge #1
This is very simple CrackMe binary. After disassemble the binary it in IDA Pro. We can clearly read from
the comment of the assembly there are two branches, one of them is output a message confirm the input key
is correct. Tracing back to the assembly we discovered the code take the input buffer and xor each char 
with 0x7D and compare to the chars stored in the data section. We located the data section in the address
of 0x00402140. Take each byte, xor them with 0x7D, the results would be the key we are looking for, the 1 challenge should be done within 10 minutes. Following code is how I obtain the email key.

```python
#!/usr/bin/env python

a = [0x1F, 0x08, 0x13, 0x13, 0x04, 0x22, 0x0E, 0x11, 
     0x4D, 0x0D, 0x18, 0x3D, 0x1B, 0x11, 0x1c, 0x0F, 0x18, 
     0x50, 0x12, 0x13, 0x53, 0x1E, 0x12, 0x10]
s =''

for aa in a:
    aa ^= 0x7d 
    s += chr(aa)

print s
```

# Challenge #2
Using IDA Pro, it is easy to find this one is similar to the Challeng #1, except it use a routine to validate
the input data in stead of simple xor operation. I identified the code block loc_4010A2 as the one we should 
focus on decoding, and the following data in the text section as the data to be decoded.
```
.text:004010E4                 db 0AFh
.text:004010E5                 db 0AAh
.text:004010E6                 db 0ADh
.text:004010E7                 db 0EBh
.text:004010E8                 db 0AEh 
.text:004010E9                 db 0AAh, 0ECh, 0A4h
.text:004010EC                 dd 0AAAEAFBAh, 0B0A7C08Ah, 0A5BA9ABCh, 0B8AFBAA5h, 0AEF9B89Dh
.text:004010EC                 dd 0BCB4AB9Dh, 9A90B3B6h, 0A8h, 3Dh dup(0)
.text:00401200                 dd 380h dup(?)
```

The below decoding program I come up is to simulate the assembly language using python. It works under the mind
of bruteforce. So the code took a while to execute and output the results: a_Little_b1t_harder_plez@flare-on.com.


```python
#!/usr/bin/env python

en = [0xAF, 0xAA, 0xAD, 0xEB, 0xAE, 0xAA, 0xEC, 0xA4, 0xBA, 0xAF, 0xAE,
      0xAA, 0x8A, 0xC0, 0xA7, 0xB0, 0xBC, 0x9A, 0xBA, 0xA5, 0xA5, 0xBA,
      0xAF, 0xB8, 0x9D, 0xB8, 0xF9, 0xAE, 0x9D, 0xAB, 0xB4, 0xBC, 0xB6,
      0xB3, 0x90, 0x9A, 0xA8]

en.reverse()

results = ""

#initialize the values here
D_EAX = 0x0018FF84 
D_ECX = 0x00000025
D_EDX = 0x0008E3C8
D_EBX = 0x00000000
D_EFLAGS = 0x00000246
D_stack =[]

for e in en:
    #update the initial value
    O_EAX = D_EAX
    O_ECX = D_ECX
    O_EDX = D_EDX
    O_EBX = D_EBX
    O_EFLAGS  = D_EFLAGS
    O_stack = D_stack
    for c in range(128):
        EAX = O_EAX       
        ECX = O_ECX       
        EDX = O_EDX       
        EBX = O_EBX       
        EFLAG  = O_EFLAGS 
        stack = O_stack

        # begining of the block
        EDX = (EDX & 0xFFFF0000) | (EBX & 0x0000FFFF)
        EDX = EDX & 0x03 
        EAX = (EAX & 0xFFFF0000) + 0x01C7
        # push eax to save it in stack for use later.
        stack.append(EAX)

        # sahf instruction, move AH to EFLGS, hard code this, because the second bit of
        # EFLAGs is default to 1
        AH = (EAX & 0x0000FF00) >> 8
        AH |= 0x2
        AH &= ~(0x1 << 3)
        AH &= ~(0x1 << 5)
        EFLAGS = AH 
        # EFLGS = (0x0000FF00 & EAX) >> 8

        # load the input byte to AL (LODS instruction)
        EAX = (EAX & 0xFFFFFF00) + c 

        # push register value to stack
        stack.append(EFLAGS)

        # xor AL with the AL of previous AL
        EAX = (EAX & 0xFFFFFF00) + ((EAX & 0x000000FF) ^ 0xC7)# which is the saved EAX = 0x001801C7

        # xchg DL and CL
        DL = EDX & 0x000000FF
        CL = ECX & 0x000000FF
        EDX = (EDX & 0xFFFFFF00) | CL # keep the invariance 
        ECX = (ECX & 0xFFFFFF00) | DL
        print "EDX = 0x%02X After xchg" % EDX 
        print "ECX = 0x%02X After xchg" % ECX 

        # ROL AH, CL
        AH = (EAX & 0x0000FF00) >> 8
        CL = ECX & 0x000000FF
        AH = AH << CL
        AH = AH << 8
    
        EAX &= ~(0xFF << 8)
        EAX = EAX + AH
        # POPFD
        EFLAGS = stack.pop()

        # ADC add AL and AH with carry bit, it depends on EAX
        EAX = (EAX & 0xFFFFFF00) + (EAX & 0x000000FF) + ((EAX & 0x0000FF00) >> 8) + 1

        # xchg DL and CL
        CL = EDX & 0x000000FF
        DL = ECX & 0x000000FF
        EDX = (EDX & 0xFFFFFF00) | DL # keep the invariance 
        ECX = (ECX & 0xFFFFFF00) | CL

        EDX ^= EDX

        # EAX and 0xFF
        EAX = EAX & 0x000000FF

        # update the value of BX
        BX = (EBX & 0x0000FFFF) + (EAX & 0x0000FFFF)
        EBX = (EBX & 0xFFFF0000) | BX 

        AL = EAX & 0x000000FF
        db = e & 0x000000FF

        EAX = stack.pop()

        if AL != db:
            DX = EDX & 0x0000FFFF
            CX = DX
            ECX = (ECX & 0xFFFF0000) | CX
        else:
            results += chr(c)
            #update the values
            D_EAX = EAX
            D_EBX = EBX
            D_ECX = ECX
            D_EDX = EDX
            D_EFLAGS  = EFLAGS 
            D_stack = stack
            break

print results, len(results)
```

## Lesson Learned
1. Too much focus on the details, should be able to work out the solution as fast as as you can in such a 
competition setting. 
2. Write code that reverse the encoding process is better than brute force. Try to develop such a capability 
that is fast enough.

# Challenge #3

This Challenge is hard in the way that I don't know how to debug the child process. Because the default ways
of debugging a process in IDA, OllyDBG, and WinDBG are only debug a single process. Until I found the the option
in WinDBG ```.childdbg 1```, the hardness level of reversing has been eased a lot. The process created a child
process, which start a python script. I found the python is resposible for bring up the windows and the image
window. 

When I obtained the file, I checked it with CFF explore and PEiD. The file is a PE32 executable and it isn't 
packed. I disassembled in IDA and see the string of the executable. I saw the string "inflate 1.2.7 Copyright
 1995-2012 Mark Adler", but not the Window title text "Look inside, you can find one there!". There are also 
no "CreateWindows" API imported. Googling it found this might be a certain proprietary file format, and it is 
possible that the code and data in it are packed. Browsing through the disassembled code found some interesting 
strings such as "%s could not be extracted!\n" exception handling code. This give me more confidence about my 
intuitions.

Running the code in IDA debugger revealed that the process create some file under the directory of 
"C:\Users\rui\AppData\Local\Temp\_MEI52922\", here is the file lists created in under the directory:
```
Mode                LastWriteTime     Length Name
----                -------------     ------ ----
-a---         8/27/2015   3:09 PM      68608 bz2.pyd
-a---         8/27/2015   3:09 PM        472 elfie.exe.manifest
-a---         8/27/2015   3:09 PM       1857 Microsoft.VC90.CRT.manifest
-a---         8/27/2015   3:09 PM     224768 msvcm90.dll
-a---         8/27/2015   3:09 PM     568832 msvcp90.dll
-a---         8/27/2015   3:09 PM     655872 msvcr90.dll
-a---         8/27/2015   3:09 PM     110592 pyside-python2.7.dll
-a---         8/27/2015   3:09 PM    1853440 PySide.QtCore.pyd
-a---         8/27/2015   3:09 PM    6947328 PySide.QtGui.pyd
-a---         8/27/2015   3:09 PM    2449920 python27.dll
-a---         8/27/2015   3:09 PM    2554880 QtCore4.dll
-a---         8/27/2015   3:09 PM    8360448 QtGui4.dll
-a---         8/27/2015   3:09 PM      10240 select.pyd
-a---         8/27/2015   3:09 PM     108544 shiboken-python2.7.dll
-a---         8/27/2015   3:09 PM     686080 unicodedata.pyd
-a---         8/27/2015   3:09 PM     358400 _hashlib.pyd
-a---         8/27/2015   3:09 PM      44544 _socket.pyd
-a---         8/27/2015   3:09 PM     899584 _ssl.pyd
```
This tell me that the elfie picture is created with some python library. My progress stuck at this step, 
because the elfie picture windows is created in a new process. Since I haven't REed child process. 
I cannot stop the debugger in the child code, there is no way for me to look inside!

Fortunately, windbg has the capability to debug child procesess. run the PE file, and suspend execution at the 
beginning of the parent process. Issue command ```.childdbg 1``` in the windbg command window. In this way, 
windbg is able to debug the child process, and let me look in to the memory and find the key. 

In the debug options, I check only the option: "suspend on process entry/exit". using the user space debugging. 
Then set a break point on the entry to _main function in the .text:0040B29B. step over the instruction step by 
step, and step into some interesting function, at certain point, I land on a interesting code block 
```
.text:004019C9 push    offset aImportSys               ; "import sys\n"
.text:004019CE call    dword_425658
.text:004019D4 push    offset aDelSys_path             ; "del sys.path[:]\n"
.text:004019D9 call    dword_425658
.text:004019DF lea     eax, [edi+2068h]
.text:004019E5 add     esp, 10h
```

The call dword_425658 is actually refer to a table that map it to a address 1E116f10, where we found the 
following interesting code block.
```
debug054:1E116F11 inc     esp
debug054:1E116F12 and     al, 4
debug054:1E116F14 push    0
debug054:1E116F16 push    eax
debug054:1E116F17 call    python27_PyRun_SimpleStringFlags
debug054:1E116F1C add     esp, 8
debug054:1E116F1F retn
```

It call a python native API which is to run python code in a file or buffer. set a break point at the call to 
the method python27_PyRun_SimpleStringFlags, eax point to the buffer that store the python code it going to 
execute. the call python27_PyRun_SimpleStringFlags run for 8 times until the goat window showup. The buffer 
content saved in the file named py0.txt to py7.txt in the dump files. The last executed buffer(py7.txt) is 
the python code that create the window and show the picture. It is moderately obfusicated, with all the variable 
using a 32 char only taken from '0' and 'o'. at the end of the buffer, it decode the base64 encoded code and 
executed the application.

I Substitute the exec() with print. It dump the de-obfuscated python code, in which we can read that most of the
string is reversed. When I browse through the code, the string elfie run into me, in fact it is not elfie, 
it is eiflE. Anyway, this string catches my eyeball and it happens to be the email address if it is interpreted
backwards ___moc.no-eralf@OOOOY.sev0000L.eiflE___. which is ___Elfie.L0000ves.YOOOO@flare-on.com___ when reversed.
 
## Lesson Learned
1. Always expand you horizon in using new tools and understand more sophisticated code obfuscation techniques.
2. Don't lost once you solving a problem that you didn't deal with before. Gradually research the related and make
sure you are not digressed and always ensure each step is some progress to the ultimate goal of solving this problem.

# Challenge #4
This challenge is a packed file. Using PEiD found packer "UPX 0.89.6 - 1.02 / 1.05 - 2.90 -> Markus & Laszlo".
PEdump.com and CFF explorer found it is packed by Crypto-Lock v2.02 (Eng) (Ryan Thian). The entrypoint is 0x0000B440.
I tried Ollydump and ImpRec tools, they are not working. when I open the CFF explore to view the header of the packed 
file. I discovered that there is a unpacker in this tool. run the unpacker has successfully unpacked the file, 
which produce the Compiler information Microsoft Visual C++ 8. 

I am very happy with it and saved the unpacked file immediately. However, this is not the end of the journey. 
The code won't run in Olly and IDA, in both debuggers, it fault at 0x323F2B, where it try to referenced 
memory at 0x407018, and the later mem address couldn't be read. Same problem in Olly.

I then turn to the following process, I compared the OEP(0x0B43A8A) of the auto unpacker identified with the 
one I found in the debugger. to see which address the instruction try to reference at the 0x323F2B.
```
00B4B621   .-E9 6484FFFF    JMP youPecks.00B43A8A  %from OllyDbg
```
I switch to read slides "Unpacking Malware, Trojans and Worms". in which I got the following key point 
in solving this problem.

1. Find the OEP by popad instruction.
2. dump the memory to disk
3. modify the IAT section of the PE file (using impRec) and this is the original 
executable (expand the Import Address Table of the previous one.
4. SEH's role in unpacking?

With further digging, I found this [post](http://www.woodmann.com/forum/showthread.php?9525-PECompact-v1-67-Delphi-DLL)
which described the same problem as I have. In the discussion, there is a link to a tutorial which 
discuss to unpack the PEcompact packed executable.

After studied youPecks_unpacked.exe, I discovered some of the base64 encoded 
strings in the binary. simple b64decode() function won't gave us a meaningful string, 
so I have to come up a python script that is able to decode the obfuscated string. 
(it turns out that this is not possible without because of the permutations of 65 
characters string will run for ever to generate.)

I have to turn to the program itself, by study its own control flow and figure out 
where to get the dam email address.

The process should go like this: I run the unpacked binary, trace it to the console print
"2 + 2 = 4". in the process, It should run some code to decode the obfuscated strings. 
From which I can figure out the indexing string and decode the mysterious string and 
see what happens there.

But I have the following questions. 
1. What's the different between two packers: "UPX 0.89.6 - 1.02 / 1.05 - 2.90 -> Markus & Laszlo"
   and "Crypto-Lock v2.02 (Eng) (Ryan Thian)".
2. How can I manually unpack the file? instead of using CFF explorer.
3. How possible that the OEP is changed every time loaded into debugger.


After the study of the above theree questions, I told myself that try to figure out the control flow and the important 
address, and save them. 

## 1st run the youPecks: (debug option: stop at entry of self-extractor)

stop at:
```
0122B440	60		PUSHAD
```
set a hardware breakpoint at the current ESP. F9 find the POPAD, and near it found the OEP 0x00223A8A. 
F7 to continue the execution. until to the 
```
00223964		CALL youPecks.00221350
```
and this time I run over the address, I start it again.

## 2nd run the youPecks:

stop at:
```
0122B440	60		PUSHAD
```
Continue to 
```
00223964		CALL youPecks.00221350
```
F7 bring us to the function itself at 00221350.

## 3rd run the youPecks:
stop at:
```
00A5BA8A	60		PUSHAD
```
set hardware break point, F9 get to
??????????

We end up at the OEP? 00A53A8A.

## 4th run, in which I run the youPecks.exe and the youPecks_unpacked.exe

|                   | break at     | Instructions |
|-------------------|--------------|------------- |
|youPecks_unpacked  | 0x00F73A8A   | CALL youPecks.00F73F23 |
|youPecks           | 0x01273A8A   | CALL youPecks.01273F23 |


F7 step into the first call instruction, at the very few instruction the unpacked
fault at address ```0x00F73F2B		MOV EAX, DWORD PTR DS:[407018]```

We see that all other memory reference are in the same section with 0x00F7XXXX, 
It try to reference address from the different section. One possibility is that
the unpacked binary might need to modify the OEP, if we put the OEP as 0x00400000,
The memory reference would not fault at 0x407018. So what I do is to 

1. check the unpacked whether it randomize the OEP, (this is happens in the youPecks.exe)
2. if OEP of unpacked is fixed, we just need to adjust it (hex editing)
3. if the OEP of unpacked is randomized as it does in the packed file, we have to find out how it works, and try to fix the address "shift" in memory reference.
		
I checked in PE Tools, and PEiD, the EP is 0x00003A8A. I then use ollydump to manual obtain the executable, I have got three files.

| 3A8A as OEP | 3847 as OEP |
|-------------------------------------|-----------------------------------------|
| youPerks-unpacked-manual.exe        | youPerks-unpacked-manual-jmp-notfix.exe |
| youPerks-unpacked-manual-fixIAT.exe | youPerks-unpacked-manual-jmp.exe        |

When I really have a hard time, I discovered that some one in the RE stackExchange pointed out 
that I can modify the binary and don't allow it to load at a random location each time it runs. 

ACTUALLY, THIS WORKED!!! The unpacked executable is working now!!!! It is just how to find
the secret from the obfuscated strings. Now I can run the unpacked binary in IDA debugger. 
When run in the IDA debugger, I have got "2 + 2 = 5". How can that happen?

To run the youPecks-rm-aslr in OllyDB and to see what happened. It actually stop randomize the 
image base each time it run in OllyDB. In this way, things becomes easier. I have use manual 
unpack (dump in OllyDbg, fix IAT using ImpRec) and CFF auto unpack. generated the following binaries.

* youPecks-rm-aslr
* youPecks-rm-aslr-unpack-CFF
* youPecks-rm-aslr-unpack-manual-IAT
* youPecks-rm-aslr-unpack-manual-IAT-inner
* youPecks-rm-aslr-unpack-manual-no-IAT
* youPecks-rm-aslr-unpack-manual-no-IAT-inner

The "inner" means I discovered the unpacked executable is still packed under 
RDG packer detector, However, they are not that different while I compare them 
in LordPE. The only difference is the entry point. Ignore it and continue looking 
for the solution.

While inspect the string of the unpacked file, I found some base64 encoded strings. 
simple decode revealed that it is not using a standard indexing string. It must be some 
customized indexing string. How to find them?

I set break point at the address that those obfuscated string accessed, however, they
are not break anyway. Also, tracing the unpacked binary suggest that the string has never 
been access in running the program. It must be some string generated
during the unpacking process, and they are not used in the unpacked program, they are just
happens to be there. It make sense that Flare team member do such kinds of tricks to fool us.

The good thing is that the address now is fixed in the 0x400000 range. and we can combine 
IDA and OllyDbg to set break point at the unpacking code. to see when those string are
generated. We can figure out how they generated and how they could be decoded.

I have loaded up the unpacked program in IDA. Load the packed program into OllyDbg.
found the memory reference to the string "abcdefghigklmnopqrstuvwxyzABCDEFG...", which
happens to be in 0x004051B8. I set a memory access break point at the address. It breaked!!!

I am happy that the routine is in the unpacking code, so just to find out the routine that
generate the base64 encoded string using the "abcdefghigklmnopqrstuvwxyzABCDEFG...".

In OllyDbg, set a memory access bp at 0x4051B8, and also set at 0x4015A9. the program will
stop at where the string will be written to. Trace through the code nearby, we can identify
the routine that the string has been generated. In this way, we can figure out how the 
strings has been generated.

Decode them will help us get the secret!!!!!!!
```
0040B458   > 8A06           MOV AL,BYTE PTR DS:[ESI]
0040B45A   . 46             INC ESI
0040B45B   . 8807           MOV BYTE PTR DS:[EDI],AL
0040B45D   . 47             INC EDI
0040B45E   > 01DB           ADD EBX,EBX
0040B460   . 75 07          JNZ SHORT youPecks.0040B469
0040B462   > 8B1E           MOV EBX,DWORD PTR DS:[ESI]
0040B464   . 83EE FC        SUB ESI,-4
0040B467   . 11DB           ADC EBX,EBX
0040B469   >^72 ED          JB SHORT youPecks.0040B458
.....
```

These code is to generate the obfuscated string. To figure out the initial value of
ESI and and EDI, then code it up in python to decode all the strings.

It seems that the base64 looking string in the unpacked binary is not encoded string.
It is moved from the packed code. (I have to verify this). Now I am going to discover
what the packer actually doing and how it generate the mysterious string and what's the purpose of it.

I start the packed program. right on start up the program it start to copy code from
the data section 0x00409000 to the 0x00401000. I need to find out how these section of
code run and how many of the chars it has copied by these section of code.
(compare to the later section that move the base64 encoded string)

The first 4 bytes from 0x409000 has been skipped. Then it is base on the value of
ebx to branching to different branch and copy the chars selectively, we should found out
what invariance it have in copying. Then we can decode the mysterious strings.

I discovered, that this is a self modifying code, Since it the output is not calculated
It is printed out in the data area as 5 and then modified it to 4 and then print after
print the string "2 + 2 =". Together with this, it also print the secrete strings.
I am feeling that those string are never used and they are the artifact that i should focus
on. I found that once POPAD executed, it will reduce ESP to EAX, and then jump to the OEP.
ESP saved: 0018FF8C before PUSHAD,

I discovered the code is generated and finally modify itself. i.e. change "5" to "4"
after write it to the memory. So I start to focus on the memory area that generate
the strings I observed in IDA.

I looked at the address 0040A58F (source the packer used to generate the code) and the
address 004051B8 (destination the packer used to write the generated code). One of the
pattern I found is that the source have four non-ascii chars embedded in the memory,
during the code generation, packer code will skip them. When I was following the code.
It become more and more complicated, it comes to my mind that those obfuscated string
might be used in the original (unpacked) binary. I have to run it in the debugger, and
check it again.

While in graphic mode, everything is clear. It is because we didn't gave any command args
the code just run a small section of the code and exit. We have to figure out where it
branched and redirect it to the part that involving the obfuscated strings.

Not far from the beginning(after print the "2 + 2 = 4" string.) we see it branched based
on the value of argc.(0x004014EE cmp [ebp = argc], 2), we gave it a command line argument
in Debugger -> Process Options. We can also modify the ZF to make it jump, however, if you
do so, you'll find that call to atoi will simply return and the following code won't continue.

From the branch we would to take, we discovered that we should provide a date as the command
line argument. I gave it 9042015, it works.

I followed the graph to find the next chain of blocks below that involving the obfuscated string.
At the code block 0x00401B99, I found the branch condition and change the ZF to go to the
desired branch. After run the chain of code blocks involving the obfuscated string. There is
a line printed in the stdout and the code exit. It just one short line, but I didn't read it.
Next, I will set a break point at where the "\n" is printed. I set a bp at address 0x00402468
and step over it. Bingo!!! Here it is.

## Lesson Learned
* The lesson I learned is always look inside the artifact instead of where you think it might
correlate with. Please try to avoid solving problems in the complicated way.


# Challenge #5
This problem is simple, somehow I expect such a problem after solving some challenge problem. 
A executable that send out traffic, a pcap file with information contained. Find out the key.
The Challenge #5 is exactly this kind of problem. while open the pcap file with Wireshark, I 
immediately discovered the data has been send out by the sender.exe. Those bytes that discovered
in the network traffic are must be encoded in some way otherwise the challenge problem is too simple.
I discovered there are strings related to base64 encoding. Try decoding the network traffic data 
gave me the unreadable code. 

I discovered the code block at loc_F811C0 is have a routine that obfuscate the input string
read from the key.txt file. Have to identify the function of the routine to decrypt the 
secrete in the traffic. Tried IDAscope, it didn't find any encryption code, also tried PEiD, 
no encryption found. Next step is to go through the manipulation code again and figure out how 
the buffer handed to the network buffer, and send out in a http connection.

There are no encryptions, I am pretty sure that it use a key "flarebearstare" to encrypt a string,
it also use base64 for encoding. But how it has been done? I didn't take the time to fully understand
the code, instead, I try the different order of base64 encoding and encryption. it turns out that 
the data captured in the network traffic is first encrypted using the key "flarebearstare" and then
using a customized base64 encoding method. Following is the decoding routine and the decryption that
produce the key.
	
	#!/usr/bin/env python
	
	import sys
	import base64
	import string
	
	alt_base64 = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789+/"
	std_base64 = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
	data = "UDYs1D7bNmdE1o3g5ms1V6RrYCVvODJF1DpxKTxAJ9xuZW=="
	key = "flarebearstare"
	string.maketrans
	data = data.translate(string.maketrans(alt_base64, std_base64))
	decode_data = base64.b64decode(data)
	#print base64.b64decode(str)
	
	result = []
	for i in range(len(decode_data)):
		result += chr(ord(decode_data[i]) - ord(key[i % 14]))
		print ''.join(result)

# Challenge #6
I didn't solve this problem at last. I am stopped here because I never worked on Android platform, and get into the ARM assembly is a really headache. However, I stick on it until the last minute and get a good idea of what looks like in Android work. Solving this challenge at the last moment is almost like a reverse engineering Android tutorial.

My though would be run the application in a emulator, see what it does. then I could decide what extent 
I want the reverse engineering to be, learning new tools or not.

## Setup Android Development Environment

1. Follow these [links](http://www.codeproject.com/Articles/797553/Setting-Up-Your-Android-Development-Environment)
2. According to the following instructions install the downloaded application
	* Execute the emulator (SDK Manager.exe->Tools->Manage AVDs...->New then Start)
	* Start the console (Windows XP), Run -> type cmd, and move to the platform-tools folder 
	  of SDK directory.
	* Paste the APK file in the 'android-sdk\tools' or 'platform-tools' folder.
	* Then type the following command.
		adb install [.apk path]
		Example:
			adb install C:\Users\Name\MyProject\build\Jorgesys.

So I run in command line:
```
D:\Program Files\Android sdk\platform-tools>adb install android.apk
* daemon not running. starting it now on port 5037 *
* daemon started successfully *
1024 KB/s (1078129 bytes in 1.028s)
        pkg: /data/local/tmp/android.apk
Failure [INSTALL_PARSE_FAILED_NO_CERTIFICATES]
```

Now I have studied the signing of android apps. Here is the steps that to sign the app
and to install it in the emulator.

## Signing Android Apps

### Step 1

```
D:\Program Files\Android sdk\platform-tools>keytool -genkey -v -keystore C:\User
s\rui\.android\debug.keystore -alias flare-android -keyalg RSA -keysize 2048 -va
lidity 20000
Enter keystore password: android
What is your first and last name?
  [Unknown]:  Harry
What is the name of your organizational unit?
  [Unknown]:  UM
What is the name of your organization?
  [Unknown]:  UM
What is the name of your City or Locality?
  [Unknown]:  US
What is the name of your State or Province?
  [Unknown]:  FL
What is the two-letter country code for this unit?
  [Unknown]:  01
Is CN=Harry, OU=UM, O=UM, L=US, ST=FL, C=01 correct?
  [no]:  yes

Generating 2,048 bit RSA key pair and self-signed certificate (SHA1withRSA) with
 a validity of 20,000 days
        for: CN=Harry, OU=UM, O=UM, L=US, ST=FL, C=01
Enter key password for <flare-android>
        (RETURN if same as keystore password):android
Re-enter new password:
[Storing C:\Users\rui\.android\debug.keystore]
```

The above information is from [here](http://stackoverflow.com/questions/18589694)

	Keystore name: "debug.keystore"
	Keystore password: "android"
	Key alias: "androiddebugkey"
	Key password: "android"
	CN: "CN=Android Debug,O=Android,C=US"

### Step 2

```
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore debug.keystore android.apk flare-android
```

This gave the following ERRORs:
-----------------------------------
D:\Program Files\Android sdk\platform-tools>jarsigner -verbose -sigalg SHA1withR
SA -digestalg SHA1 -keystore C:\Users\rui\.android\debug.keystore -storepass and
roid -keypass android android.apk androiddebugkey
jarsigner: unable to sign jar: java.util.zip.ZipException: invalid entry compres
sed size (expected 9227 but got 9389 bytes)

A search found that it is because the apk file has already been signed.
We remove the signature and resign using our debug.keystore.
http://stackoverflow.com/questions/5089042/

Step 1:
```
D:\Program Files\Android sdk\platform-tools>zip -d android.apk META-INF/*
deleting: META-INF/MANIFEST.MF
deleting: META-INF/CERT.SF
deleting: META-INF/CERT.RSA
```
Step 2:
```
D:\Program Files\Android sdk\platform-tools>jarsigner -verbose -sigalg SHA1withR
SA -digestalg SHA1 -keystore C:\Users\rui\.android\debug.keystore -storepass and
roid -keypass android android.apk androiddebugkey
```
See the reference at [this link](http://developer.android.com/tools/publishing/app-signing.html)
Step 3:
```
D:\Program Files\Android sdk\platform-tools>adb install android.apk
1127 KB/s (1078782 bytes in 0.934s)
        pkg: /data/local/tmp/android.apk
Success
```
OK, we now have the app installed in the emulator.

Run the app show that it is a android version of password checking application.
We can decompile the code to java, then problem solved.

Let's do it!!!

first change the postfix of the apk file to zip, extract it in the folder android

Download the dex2jar and run the following command based on the official user 
[guide](http://sourceforge.net/p/dex2jar/wiki/UserGuide/)

```
D:\Dropbox\hacking\Tools\dex2jar-2.0>d2j-dex2jar.bat D:\Dropbox\hacking\Flare-Ch
allenge\2015\C6\android\classes.dex
dex2jar D:\Dropbox\hacking\Flare-Challenge\2015\C6\android\classes.dex -> .\clas
ses-dex2jar.jar
```

copy the classes-dex2jar to the C6 directory.download the JD_GUI and decompile the classes-dex2jar.
After reference to the official android site, I found that we should locate the key 
from the shared library libvalidate.so in the lib directory of the unzipped file.

Open it in IDA Pro, I found the string "That's it!". This should be where the correct
email address lead to. I have to figure out what those arm code is doing.

I need to go through this tutorial
http://blog.dornea.nu/2015/07/01/debugging-android-native-shared-libraries/
http://code.tutsplus.com/tutorials/advanced-android-getting-started-with-the-ndk--mobile-2152
and familiar with the Android NDK.

## Lesson Learned
1. To become ridiculously aggressive about your goal. Someone took 27 continuous hours to solve 
all the 11 problems. Think about it.
2. Always reaching out for help, don't reinvent the wheel alone.
3. Practice makes perfect!
