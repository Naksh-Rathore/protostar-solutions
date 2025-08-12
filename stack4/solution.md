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

*Again*, you may be asking how do we use this to go the the `win()` function. The answer lies in **overflowing the stack values until we reach the return address, then
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

**TO DO**

## Resources

* [C Code](https://exploit.education/protostar/stack-four/)
* [Geeks for Geeks Article on Return Addresses](https://www.geeksforgeeks.org/dsa/how-is-return-address-specified-in-stack/)


