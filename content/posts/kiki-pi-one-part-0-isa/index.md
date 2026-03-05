---
date: '2026-02-22'
draft: false
series:
- KiKi-Pi-One
series_order: 0
tags:
- kiki-pi-one
- cpu-design
- computer-architecture
- hardware
- isa
title: 'Part 0: The Instruction Set Architecture - Designing KiKi-Pi-One''s Language'
---

*This is Part 0 of the KiKi-Pi-One series, where we build a 16-bit CPU from scratch.*
*[GitHub](https://github.com/SreejitS/KiKi-Pi-One)*

Before writing a single line of hardware code, we need to answer one question: **what instructions should our CPU understand?**

The Instruction Set Architecture (ISA) is the contract between hardware and software. Every component we build - the ALU, the registers, the memory, and the assembler - derives from this spec. Get the ISA wrong, and everything downstream breaks.

KiKi-Pi-One uses exactly **two instruction types**. That is not a limitation. With just two instructions, we can understand every bit, build the hardware without a lookup table, and implement a complete assembler in under 200 lines of Python. The architecture is inspired by the Nand2Tetris HACK computer - one of the most elegant minimal ISAs ever designed.

## The Big Picture

Every program is a sequence of 16-bit binary words stored in ROM. The CPU fetches one word per clock cycle and executes it. There are two kinds:

```
0 vvvvvvvvvvvvvvv   <- A-instruction (bit 15 = 0)
111accccccdddjjj    <- C-instruction (bit 15 = 1)
```

That is it. Let us look at each one.

## The A-Instruction: Loading a Value

The A-instruction loads a 15-bit constant into the **A register**.

```
Bit:  15  14 13 12 11 10  9  8  7  6  5  4  3  2  1  0
       0   v  v  v  v  v  v  v  v  v  v  v  v  v  v  v
       |   +------------- 15-bit value --------------+
     fixed
```

Bit 15 is always 0. The remaining 15 bits hold any value from 0 to 32,767.

In assembly, you write it with an `@` prefix:

```asm
@5       // loads 5 into A
@100     // loads 100 into A
@SCREEN  // loads 16384 into A (predefined symbol)
```

The A register serves double duty:
1. **Data register** - holds a value the ALU can use
2. **Address register** - tells the CPU where in RAM to read or write (`M = RAM[A]`)

Any time you want to work with a memory address or a large constant, you first load it into A.

## The C-Instruction: Compute, Store, Jump

The C-instruction is where all the work happens. It tells the CPU to:

1. **Compute** something using the ALU
2. **Store** the result somewhere
3. **Maybe jump** to a different instruction

```
Bit:  15  14  13  12  11  10   9   8   7   6   5   4   3   2   1   0
       1   1   1   a   c1  c2  c3  c4  c5  c6  d1  d2  d3  j1  j2  j3
       |       |   |   |                    |   |        |   |        |
       +-fixed-+   a   +------- comp -------+   +- dest -+   +- jump -+
```

Bits 14-13 are always 1 (reserved). The remaining bits split into three fields.

### The `comp` field (bits 12-6): what to compute

Seven bits control the ALU. Bit 12 (`a`) selects whether the second ALU input comes from the **A register** or **M (RAM[A])**. The other six bits select the operation.

| Mnemonic | Bits (a cccccc) | Result |
|---|---|---|
| `0`    | `0 101010` | Constant 0 |
| `1`    | `0 111111` | Constant 1 |
| `-1`   | `0 111010` | Constant -1 |
| `D`    | `0 001100` | D register |
| `A`    | `0 110000` | A register |
| `M`    | `1 110000` | RAM[A] |
| `!D`   | `0 001101` | NOT D |
| `!A`   | `0 110001` | NOT A |
| `!M`   | `1 110001` | NOT M |
| `-D`   | `0 001111` | Negate D |
| `-A`   | `0 110011` | Negate A |
| `-M`   | `1 110011` | Negate M |
| `D+1`  | `0 011111` | D plus 1 |
| `A+1`  | `0 110111` | A plus 1 |
| `M+1`  | `1 110111` | M plus 1 |
| `D-1`  | `0 001110` | D minus 1 |
| `A-1`  | `0 110010` | A minus 1 |
| `M-1`  | `1 110010` | M minus 1 |
| `D+A`  | `0 000010` | D plus A |
| `D+M`  | `1 000010` | D plus M |
| `D-A`  | `0 010011` | D minus A |
| `D-M`  | `1 010011` | D minus M |
| `A-D`  | `0 000111` | A minus D |
| `M-D`  | `1 000111` | M minus D |
| `D&A`  | `0 000000` | D AND A |
| `D&M`  | `1 000000` | D AND M |
| `D\|A` | `0 010101` | D OR A |
| `D\|M` | `1 010101` | D OR M |

28 operations from 6 bits. The trick: the 6 control bits apply successive transformations to the inputs - zero it, negate it, add or AND, negate the output. Different combinations give you all 28 useful operations. We explore this in detail in [Part 2 (the ALU)](/posts/kiki-pi-one-part-2-alu/).

### The `dest` field (bits 5-3): where to store the result

Three bits, eight possible destinations. Multiple can be active at once:

| Mnemonic | Bits | Writes to |
|---|---|---|
| `null` | `000` | Nowhere (result discarded) |
| `M`    | `001` | RAM[A] |
| `D`    | `010` | D register |
| `MD`   | `011` | RAM[A] and D |
| `A`    | `100` | A register |
| `AM`   | `101` | A register and RAM[A] |
| `AD`   | `110` | A register and D |
| `AMD`  | `111` | A register, RAM[A], and D |

### The `jump` field (bits 2-0): when to jump

After the ALU computes its result, the CPU checks the jump condition. If true, the program counter (PC) jumps to `ROM[A]`. If false, it increments.

| Mnemonic | Bits | Condition |
|---|---|---|
| `null` | `000` | Never jump |
| `JGT`  | `001` | Jump if result > 0 |
| `JEQ`  | `010` | Jump if result = 0 |
| `JGE`  | `011` | Jump if result >= 0 |
| `JLT`  | `100` | Jump if result < 0 |
| `JNE`  | `101` | Jump if result != 0 |
| `JLE`  | `110` | Jump if result <= 0 |
| `JMP`  | `111` | Always jump |

### Putting it together: reading a C-instruction

Let us decode `D=D+A` by hand:

```
Assembly:  D = D+A
           |   |__|
           |   comp (no jump)
           dest

dest = D   -> d1=0, d2=1, d3=0 -> 010
comp = D+A -> a=0, cccccc=000010
jump = null -> 000

Full instruction:
  1  1  1  0  0  0  0  0  1  0  0  1  0  0  0  0
  |  |  |  |  |              |  |     |  |     |
  C  1  1  a  +---- comp ----+  +-dst-+  +-jmp-+
```

Binary: `1110000010010000` = `0xE090`

## Memory Map

```
+--------------------------------------+
| 0x0000 - 0x3FFF |  Data RAM (16K)    |
+--------------------------------------+
| 0x4000 - 0x5FFF |  Screen Buffer     |  <- each bit = 1 pixel
+--------------------------------------+
|       0x6000    |  Keyboard          |  <- ASCII code of last key
+--------------------------------------+
```

The screen buffer is 8,192 words of 16 bits = 131,072 pixels = 256 rows x 512 columns. Writing a 1 to a bit turns the corresponding pixel black.

## Predefined Symbols

The assembler recognises these names without you defining them:

```
R0-R15    -> RAM addresses 0-15   (virtual registers)
SP        -> 0   (stack pointer)
LCL       -> 1   (local variable base)
ARG       -> 2   (argument base)
THIS      -> 3   (object pointer)
THAT      -> 4   (array/string pointer)
SCREEN    -> 16384
KBD       -> 24576
```

## A Real Program: Adding Two Numbers

Let us write a program that computes `R2 = R0 + R1` and annotate every bit:

```asm
@R0       // 0 000 000 000 000 000   -> A = 0 (address of R0)
D=M       // 1 111 110 000 010 000   -> D = RAM[0]
@R1       // 0 000 000 000 000 001   -> A = 1
D=D+M     // 1 111 000 010 010 000   -> D = D + RAM[1]
@R2       // 0 000 000 000 000 010   -> A = 2
M=D       // 1 110 001 100 001 000   -> RAM[2] = D

// Halt: infinite loop
@6        // 0 000 000 000 000 110   -> A = 6 (this line's ROM address)
0;JMP     // 1 110 101 010 000 111   -> jump to ROM[6] forever
```

Eight instructions. No ambiguity. Every bit has a purpose.

## What the ISA Gives Us

With just two instruction types, we can:
- Load any 15-bit constant into A
- Compute any of 28 arithmetic/logic operations
- Write to A, D, or RAM[A] (or all three simultaneously)
- Jump conditionally or unconditionally
- Access any of 65,536 memory addresses
- Read the keyboard and write to the screen

That is enough to implement a full assembler, a VM translator, and eventually a high-level language compiler.

The rest of the series is building the hardware that executes this spec.

## What's Next

In **Part 1**, we build the most fundamental storage element: the **register**. It is a 16-bit memory cell with one job - hold a value until told to update it.

[Part 1: The Register ->](/posts/kiki-pi-one-part-1-registers/)

---

*[KiKi-Pi-One on GitHub](https://github.com/SreejitS/KiKi-Pi-One)*
*[Full ISA Reference](/posts/kiki-pi-one-part-0-isa/)*
