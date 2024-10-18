# 1. Data movement 

| Instruction | Description                                  | Example                                   |
| ----------- | -------------------------------------------- | ----------------------------------------- |
| `mov`       | Move data or load immediate data             | `mov rax, 1` -> `rax = 1`                 |
| `lea`       | Load an address pointing to the value        | `lea rax, [rsp+5]` -> `rax = rsp+5`       |
| `xchg`      | Swap data between two registers or addresses | `xchg rax, rbx` -> `rax = rbx, rbx = rax` |

```assembly
mov rax,rsp      #rax=rsp
mov rax,[rsp]    #rax=*rsp
mov [rax],rsp    #*rax=rsp
mov [rax],[rsp]  #*rax=*rsp

mov rax,rsp+5    #invalid
lea rax,rsp+5    #invalid
```

# 2. Arithmetic Instructions

## 2.1 Unary instruction

| Instruction | Description    | Example                                         |
| ----------- | -------------- | ----------------------------------------------- |
| `inc`       | Increment by 1 | `inc rax` -> `rax++` or `rax += 1` -> `rax = 2` |
| `dec`       | Decrement by 1 | `dec rax` -> `rax--` or `rax -= 1` -> `rax = 0` |

## 2.2 Binary instruction

| Instruction | Description                                                | Example                           |
| ----------- | ---------------------------------------------------------- | --------------------------------- |
| `add`       | Add both operands                                          | `add rax, rbx` -> `rax = rax+rbx` |
| `sub`       | Subtract Source from Destination (*i.e `rax = rax - rbx`*) | `sub rax, rbx` -> `rax = rax-rbx` |
| `imul`      | Multiply both operands                                     | `imul rax, rbx` -> `rax =rax*rbx` |

## 2.3 Bitwise Instructions

| Instruction | Description                                                  | Example                             |
| ----------- | ------------------------------------------------------------ | ----------------------------------- |
| `not`       | Bitwise NOT (invert all bits, 0->1 and 1->0)                 | `not rax` -> `rax=!rax`             |
| `and`       | Bitwise AND (if both bits are 1 -> 1, if bits are different -> 0) | `and rax, rbx` -> `rax AND rbx`     |
| `or`        | Bitwise OR (if either bit is 1 -> 1, if both are 0 -> 0)     | `or rax, rbx` -> `rax=rax OR rbx`   |
| `xor`       | Bitwise XOR (if bits are the same -> 0, if bits are different -> 1) | `xor rax, rbx` -> `rax=rax XOR rbx` |

# 3.Control flow instruction

## 3.1 loop

```assembly
exampleLoop:
    instruction 1
    instruction 2
    instruction 3
    instruction 4
    instruction 5
    loop exampleLoop
```

## 3.2 Unconditional Branching

| **Instruction** | **Description**                                | **Example** |
| --------------- | ---------------------------------------------- | ----------- |
| `jmp`           | Jumps to specified label, address, or location | `jmp loop`  |

## 3.3 conditional Branching

| **Instruction** | **Condition** | **Description**                                    |
| --------------- | ------------- | -------------------------------------------------- |
| `jz`            | `D = 0`       | Destination `equal to Zero`                        |
| `jnz`           | `D != 0`      | Destination `Not equal to Zero`                    |
| `js`(sign)      | `D < 0`       | Destination `is Negative`                          |
| `jns`           | `D >= 0`      | Destination `is Not Negative` (i.e. 0 or positive) |
| `jg`(greater)   | `D > S`       | Destination `Greater than` Source                  |
| `jge`           | `D >= S`      | Destination `Greater than or Equal` Source         |
| `jl`(less)      | `D < S`       | Destination `Less than` Source                     |
| `jle`           | `D <= S`      | Destination `Less than or Equal` Source            |

Conditional instructions are not restricted to `jmp` instructions only but are also used with other assembly instructions for conditional use as well, like the `CMOVcc` and `SETcc` instructions.

For example, if we wanted to perform a `mov rax, rbx` instruction, but only if the condition is `= 0`, then we can use the `CMOVcc` or `conditional mov` instruction, such as `cmovz rax, rbx` instruction. Similarly, if we wanted to move if the condition is `<`, then we can use the `cmovl rax, rbx` instruction, and so on for other conditions. The same applies to the `set` instruction, which sets the operand's byte to `1` if the condition is met or `0` otherwise. An example of this is `setz rax`.

### RFLAGS Register

The `RFLAGS` register consists of 64-bits like any other register. However, this register does not hold values but holds flag bits instead. Each bit 'or set of bits' turns to `1` or `0` depending on the value of the last instruction.

| Bit(s)              | 0                | 1          | 2                | 3          | 4                    | 5          | 6                | 7                | 8         | 9                | 10               | 11               | 12-13               | 14          | 15         | 16          | 17               | 18                               | 19                     | 20                        | 21                  | 22-63      |
| ------------------- | ---------------- | ---------- | ---------------- | ---------- | -------------------- | ---------- | ---------------- | ---------------- | --------- | ---------------- | ---------------- | ---------------- | ------------------- | ----------- | ---------- | ----------- | ---------------- | -------------------------------- | ---------------------- | ------------------------- | ------------------- | ---------- |
| **Label** (`1`/`0`) | `CF` (`CY`/`NC`) | `1`        | `PF` (`PE`/`PO`) | `0`        | `AF` (`AC`/`NA`)     | `0`        | `ZF` (`ZR`/`NZ`) | `SF` (`NC`/`PL`) | `TF`      | `IF` (`EL`/`DI`) | `DF` (`DN`/`UP`) | `OF` (`OV`/`NV`) | `IOPL`              | `NT`        | `0`        | `RF`        | `VM`             | `AC`                             | `VIF`                  | `VIP`                     | `ID`                | `0`        |
| **Description**     | Carry Flag       | *Reserved* | Parity Flag      | *Reserved* | Auxiliary Carry Flag | *Reserved* | Zero Flag        | Sign Flag        | Trap Flag | Interrupt Flag   | Direction Flag   | Overflow Flag    | I/O Privilege Level | Nested Task | *Reserved* | Resume Flag | Virtual-x86 Mode | Alignment Check / Access Control | Virtual Interrupt Flag | Virtual Interrupt Pending | Identification Flag | *Reserved* |

Just like other registers, the 64-bit `RFLAGS` register has a 32-bit sub-register called `EFLAGS`, and a 16-bit sub-register called `FLAGS`, which holds the most significant flags we may encounter.
The flags we would mostly be interested in are:

- The Carry Flag `CF`: Indicates whether we have a float.
- The Parity Flag `PF`: Indicates whether a number is odd or even.
- The Zero Flag `ZF`: Indicates whether a number is zero.
- The Sign Flag `SF`: Indicates whether a register is negative.

### cmp

| **Instruction** | **Description**                                              | **Example**                        |
| --------------- | ------------------------------------------------------------ | ---------------------------------- |
| `cmp`           | Sets `RFLAGS` by subtracting second operand from first operand (i.e. first - second) | `cmp rax, rbx` -> `rax =rax - rbx` |