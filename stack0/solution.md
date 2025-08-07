# Stack0 Solution

## C code 

This level introduces the concept that memory can be accessed outside of its allocated region, how the stack variables are laid out, and that modifying outside of the allocated memory can modify program execution.

This level is at /opt/protostar/bin/stack0

```> [!CAUTION]
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  modified = 0;
  gets(buffer);

  if(modified != 0) {
      printf("you have changed the 'modified' variable\n");
  } else {
      printf("Try again?\n");
  }
}
```


## Buffer Overflow Attack

**A buffer overflow attack is when a program writes data beyond the buffer's allocated memory.** This can lead to things like variable overwrite and control flow hijacking. The way program overwrite too much memory is usually from things like **user input and string copying** with commands such as `gets()` and `strcpy()`. `gets()` and `strcpy()` **write as much memory as they are given, making them prone to these attacks**. You can **mitigate these attacks by using bounds checking** and using commands like `fgets()` and `strncpy()` **which make sure that the all the bytes can be held properly by the buffer**.

## Exploit

Now you may be wondering, how does a buffer overflow attack correlate to this attack? Good question, but first let's talk about the program.

This C program has a `char buffer[64]`, and a `volatile int modified`. Then it gets user input by means of the `get()` function, which is **vulnerable to a buffer overflow attack**. Lastly it checks if `modified` is not zero, and if it is you lose! 

Now again you may be thinking how will modified be not zero? It is never set to anything but zero. That is where the buffer overflow attack comes in! **If we can fill up** `buffer`, `modified` **is right behind it on the stack**, so now we have access to change `modified` and complete the CTF!

Ok, let's try it! So first of all, `buffer` holds 64 bytes. So we have to enter **64 bytes of input, then whatever we what** `modified` **to be after**.

The program should look something like this (save it in `/tmp`):

```python
padding = "A" * 64
modified_val = "1" # Change to whatever you want

print(padding + modified_val)
```


Then, to exploit run this: `python /tmp/exploit.py | /opt/protostar/bin/stack0`

Result: `you have changed the 'modified' variable` 

## Resources

* (C Code)[https://exploit.education/protostar/stack-zero/]
* (Video Walkthrough from LiveOverflow)[https://www.youtube.com/watch?v=T03idxny9jE&ab_channel=LiveOverflow]
* (Wikipedia Article on Buffer Overflow)[https://en.wikipedia.org/wiki/Buffer_overflow]
* (Wikipedia Article on the Stack)[https://en.wikipedia.org/wiki/Stack-based_memory_allocation]
