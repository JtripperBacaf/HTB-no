# 1. Shellcodes

We will use `pwntools` to assemble and disassemble our machine code, as it is an essential tool for Binary Exploitation, and this is an excellent opportunity to start learning it. 

```bash
pip install pwntools
pwn asm 'push rax'  -c 'amd64'
pwn disasm '50' -c 'amd64'
```

## 1.1 Extract shellcode

assume we have a assembly code like this:

```assembly
global _start

section .data
    message db "Hello HTB Academy!"

section .text
_start:
    mov rsi, message
    mov rdi, 1
    mov rdx, 18
    mov rax, 1
    syscall

    mov rax, 60
    mov rdi, 0
    syscall
```

We want to extract the code from it use as shellcode. So we can do following:

```python
#!/usr/bin/python3

import sys
from pwn import *

context(os="linux", arch="amd64", log_level="error")

file = ELF(sys.argv[1])
shellcode = file.section(".text")
print(shellcode.hex())
```

```bash
python shellcode.py helloHTB
```

In above code extract the `.text` part of the helloHTB.

With out Python, we can just use `objdump` to obstract it:

```bash
#!/bin/bash

for i in $(objdump -d $1 |grep "^ " |cut -f2); do echo -n $i; done; echo;
```

## 1.2 Loading shellcode

```python
#!/usr/bin/python3

import sys
from pwn import *

context(os="linux", arch="amd64", log_level="error")

run_shellcode(unhex(sys.argv[1])).interactive()
```

Then when we run like this:

```bash
python3 loader.py '4831db66bb79215348bb422041636164656d5348bb48656c6c6f204854534889e64831c0b0014831ff40b7014831d2b2120f054831c0043c4030ff0f05'
```

## 1.3 Debugging shellcode

### python

```python
#!/usr/bin/python3

import sys, os, stat
from pwn import *

context(os="linux", arch="amd64", log_level="error")

ELF.from_bytes(unhex(sys.argv[1])).save(sys.argv[2])
os.chmod(sys.argv[2], stat.S_IEXEC)
```

```bash
 python assembler.py '4831db66bb79215348bb422041636164656d5348bb48656c6c6f204854534889e64831c0b0014831ff40b7014831d2b2120f054831c0043c4030ff0f05' 'helloworld'
```

### C

```c
#include <stdio.h>

int main()
{
    int (*ret)() = (int (*)()) "\x48\x31\xdb\x66\xbb\...SNIP...\x3c\x40\x30\xff\x0f\x05";
    ret();
}
```



# 2. shellcode Techniques

## 2.1 Shellcode requirement

1. Does not contain variables
2. Does not refer to direct memory addresses
3. Does not contain any NULL bytes `00`

We will use helloHTB for example to fix each requirement

>    0:    48 be 00 20 40 00 00 00 00 00    movabs rsi,  0x402000
>    a:    bf 01 00 00 00           mov    edi,  0x1
>    f:    ba 12 00 00 00           mov    edx,  0x12
>   14:    b8 01 00 00 00           mov    eax,  0x1
>   19:    0f 05                    syscall
>   1b:    b8 3c 00 00 00           mov    eax,  0x3c
>   20:    bf 00 00 00 00           mov    edi,  0x0
>   25:    0f 05                    syscall
>
> ```assembly
> global _start
> 
> section .data
>     message db "Hello HTB Academy!"
> 
> section .text
> _start:
>     mov rsi, message
>     mov rdi, 1
>     mov rdx, 18
>     mov rax, 1
>     syscall
> 
>     mov rax, 60
>     mov rdi, 0
>     syscall
> ```

## 2.2 Remove variables

A shellcode is expected to be directly executable once loaded into memory, without loading data from other memory segments, like `.data` or `.bss`. This is because the `text` memory segments are not `writable`, so we cannot write any variables. In contrast, the `data` segment is not executable, so we cannot write executable code.

So, to execute our shellcode, we must load it in the `text` memory segment and lose the ability to write any variables. `Hence, our entire shellcode must be under '.text' in the assembly code.`

> Note: Some older shellcoding techniques (like the jmp-call-pop technique) no longer work with modern memory protections, as many of them rely on writing variables to the `text` memory segment, which, as we just discussed, is no longer possible.

There are many techniques we can use to avoid using variables, like:

1. Moving immediate strings to registers

   For the short strings,we can just move it it the registers

   ```assembly
   mov rdi 'Academy!'
   ```

   However, a register can only hold 8 byte.For long string we should use method 2

2. Pushing strings to the Stack, and then use them

   The first method is push the string to the stack and `mov` the `rsp` to `rdi` instead:

   ``` assembly
       push 'y!'
       push 'B Academ'
       push 'Hello HT'
       mov rsi, rsp
   ```

   However,`push` can not push the string value immediately,so we should load it at `rbx` and push rbx instead:

   ```assembly
       mov rbx, 'y!'
       push rbx
       mov rbx, 'B Academ'
       push rbx
       mov rbx, 'Hello HT'
       push rbx
       mov rsi, rsp
   ```

   > Note: Whenever we push a string to the stack, we have to push a `00` before it to terminate the string. However, we don't have to worry about that in this case, since we can specify the print length for the `write` syscall.

## 2.3 Remove Addresses

If we ever had any calls or references to direct memory addresses, we can fix that by:

1. Replacing with calls to labels or rip-relative addresses (for `calls` and `loops`)
2. Push to the Stack and use `rsp` as the address (for `mov` and other assembly instructions)

## 2.4 Remove NULL

NULL characters (or `0x00`) are used as string terminators in assembly and machine code, and so if they are encountered, they will cause issues and may lead the program to terminate early. So, we must ensure that our shellcode does not contain any NULL bytes `00`. If we go back to our `Hello World` shellcode disasembly, we noticed many red `00` in it:

This commonly happens when moving a small integer into a large register, so the integer gets padded with an extra `00` to fit the larger register's size.

For example, in our code above, when we use `mov rax, 1`, it will be moving `00 00 00 01` into `rax`, such that the number size would match the register size. We can see this when we assemble the above instruction:

 ```bash
 Bacaf@htb[/htb]$ pwn asm 'mov rax, 1' -c 'amd64'
 
 48c7c001000000
 ```

To avoid having these NULL bytes, `we must use registers that match our data size.` For the previous example, we can use the more efficient instruction `mov al, 1`, as we have been learning throughout the module. However, before we do so, we must first zero out the `rax` register with `xor rax, rax`, to ensure our data does not get mixed with older data. Let's see the shellcode for both of these instructions:

```assembly
xor rax,rax
mov al,1
```

We can start with the new instruction we added earlier, `mov rbx, 'y!'`. We see that this instruction is moving 2-bytes into an 8-byte register. So, to fix it, we will first zero-out `rbx`, and then use the 2-byte (i.e. 16-bit) register `bx`, as follows:

```assembly
xor rbx,rbx
mov bx,"y!"
```

Check if have NULL byte:

```python
print("%d bytes - Found NULL byte" % len(shellcode)) if [i for i in shellcode if i == 0] else print("%d bytes - No NULL bytes" % len(shellcode))
```





