# String and Pointer Representation on the 48-bit BESM-6 Architecture

> [!NOTE]
> This was written by an LLM, with almost no human review.
> -- pasko@

An architectural and implementation analysis of C strings, pointer representations, and byte manipulation on the Soviet **BESM-6** (БЭСМ-6) mainframe, based on the open-source restoration projects: [c-compiler](https://github.com/besm6/c-compiler), [v7besm](https://github.com/besm6/v7besm) (Seventh Edition Unix port), and [simh](https://github.com/besm6/simh) (hardware simulator).

---

## Executive Summary

The BESM-6 is a 1960s Soviet supercomputer built around a **48-bit word** and a **15-bit word-addressable** memory model (32,768 words of RAM). Unlike modern byte-addressed architectures (x86_64, ARM, RISC-V), the BESM-6 hardware has **no byte-level load or store instructions**. Every memory access reads or writes an entire 48-bit word.

When porting C11 and Unix Version 7 to this machine, developers face a fundamental challenge: how to represent 8-bit `char` types, strings, and byte pointers without wasting 40 bits per character or burdening C programmers with manual bit-shifting loops.

The solution is an elegant combination of **packed 6-character words** and **"Fat Pointers"** that exploit a unique hardware floating-point shift instruction (`ASX`) to achieve $O(1)$ byte extraction in just four machine instructions.

---

## 1. Character and String Representation

On the BESM-6 C compiler:
* **`CHAR_BIT` is 8**: A `char` is an 8-bit unsigned integer (`0` to `255`).
* **Word Size**: 1 word = 48 bits = 6 logical bytes (six char-units).
* **`sizeof` in Char-Units**: In C, `sizeof` returns the size of a type in *char-units* (bytes). By C language definition (C11 §6.5.3.4), `sizeof(char)` is always exactly `1` (one 8-bit char-unit). Because six 8-bit char-units fit into a single 48-bit machine word, **1 machine word = 6 char-units**. Consequently, every scalar type that occupies one full 48-bit word (`short`, `int`, `long`, `float`, `double`, `void*`) has a C `sizeof` of **`6`**!

> [!NOTE]
> **Why isn't `sizeof(int) == 1`?** If the compiler defined `sizeof(int) == 1` (meaning 1 word = 1 char-unit), then `sizeof(char)` would also be `1` (48 bits), and `CHAR_BIT` would have to be `48`. With 48-bit characters, text strings could not be packed; an array like `char str[10]` would consume 10 full words (480 bits), wasting 83% of memory! To enable packed 8-bit text strings, the compiler sets `CHAR_BIT = 8` and makes byte pointers address individual 8-bit bytes. As a result, a 48-bit word is measured as 6 char-units, making `sizeof(int) == 6`.
> 
> When you declare a standalone scalar `char x;`, C requires `sizeof(x) == 1` (1 char-unit = 8 bits). Physically in memory, the BESM-6 hardware must allocate a full 48-bit word for `x` (because RAM is word-addressable), placing the 8-bit character in the lowest byte and wasting the upper 40 bits of that word. But as far as the C `sizeof` operator is concerned, `sizeof(char) == 1` (1 byte), while `sizeof(int) == 6` (6 bytes, which is 1 word).

### Packed String Layout
To avoid wasting 83% of memory, characters in arrays and string literals are **packed six per word** in big-endian order from left to right (Byte #0 is the most significant byte; Byte #5 is the least significant byte):

```
Bit:  48   41 40   33 32   25 24   17 16    9 8     1
     ┌───────┬───────┬───────┬───────┬───────┬───────┐
     │Byte #0│Byte #1│Byte #2│Byte #3│Byte #4│Byte #5│
     └───────┴───────┴───────┴───────┴───────┴───────┘
     │  MSB  │       │       │       │       │  LSB  │
```

An array of $N$ characters occupies $\lceil N/6 \rceil$ words in memory, terminated by a NUL (`0x00`) byte. A standalone scalar `char` variable is stored in Byte #5 (bits 8-1), with bits 48-9 zeroed.

---

## 2. Pointer Representation in C Code

Because memory addresses point to 48-bit words rather than individual bytes, the C ABI defines two distinct pointer layouts within a standard 48-bit word:

```
Regular Pointer (int*, double*, struct Foo*):
Bit:  48                          16 15             1
     ┌──────────────────────────────┬────────────────┐
     │          Zero (33 bits)      │  Word Address  │
     └──────────────────────────────┴────────────────┘

Fat Pointer (char*, void*, unsigned char*):
Bit:  48 47  45 44  42            16 15             1
     ┌──┬──────┬──────┬─────────────┬────────────────┐
     │ 1│Offset│  0   │    Zero     │  Word Address  │
     └──┴──────┴──────┴─────────────┴────────────────┘
     │Tag│ 3-bit│
```

### Regular Pointers
Pointers to word-aligned scalar types or structs contain the **15-bit word address** in bits 15-1, with bits 48-16 set to zero. Pointer arithmetic (`p++`) increments the word address by 1.

### Fat Pointers (Byte Pointers)
To point to an arbitrary byte within a packed word, `char*` and `void*` are formatted as **Fat Pointers**:
* **Bit 48 (Tag)**: Always set to `1` to distinguish a fat pointer from a regular pointer (where bit 48 is `0`).
* **Bits 47-45 (Byte Offset Code, `offset_enc`)**: A 3-bit code indicating which of the six bytes is being addressed, encoded as the shift distance in bytes from the LSB:
  * `5` (`101₂`): Byte #0 (MSB, bits 48-41)
  * `4` (`100₂`): Byte #1 (bits 40-33)
  * `3` (`011₂`): Byte #2 (bits 32-25)
  * `2` (`010₂`): Byte #3 (bits 24-17)
  * `1` (`001₂`): Byte #4 (bits 16-9)
  * `0` (`000₂`): Byte #5 (LSB, bits 8-1)
* **Bits 44-42**: Always zero.
* **Bits 15-1**: The 15-bit word address in RAM.

> [!NOTE]
> Why are bits 44-42 zero? In the BESM-6 hardware word layout, bits 48-42 represent a 7-bit biased floating-point exponent. Notice that when bit 48 is `1` and bits 44-42 are `0`, the numerical value of the 7-bit field (bits 48-42) is exactly $`64 + \text{offset\_enc} \times 8`$. This clever bit alignment is the key to ultra-fast byte extraction.

---

## 3. How `strcmp` Works Without "Messy" Loops

When working on a machine that reads 6 bytes at a time, one might expect string functions like `strcmp` or `strlen` to require complex, word-unpacking assembly loops to scan for NUL terminators.

Remarkably, in `c-compiler/libc/besm6/strcmp.c`, the implementation is written in clean, textbook C:

```c
int strcmp(const char *s1, const char *s2)
{
    const unsigned char *a = (unsigned char *)s1;
    const unsigned char *b = (unsigned char *)s2;
    while (*a != 0 && *a == *b) {
        a++;
        b++;
    }
    return (int)*a - (int)*b;
}
```

### The Secret: Hardware-Assisted Lowering
The "messiness" of sub-word addressing is completely abstracted by the compiler's code generator and lightweight runtime library. When the compiler lowers `*a` (byte dereference) and `a++` (pointer increment), it generates specialized sequences:

#### 1. Byte Dereference (`*a`) in 4 Machine Instructions
The BESM-6 instruction set includes **`ASX`** (*Accumulator Shift by Exponent*), a floating-point instruction that right-shifts the accumulator by the integer value in the memory operand's exponent field minus 64.

Because a fat pointer's exponent field is pre-configured to hold $`64 + \text{offset\_enc} \times 8`$, the shift distance calculated by hardware is exactly $`\text{offset\_enc} \times 8`$ bits! Extracting a byte compiles to:

```madlen
    WTC ptr        ; Load word address from lower 15 bits of fat pointer into index register M
    XTA            ; A = mem[M] (load the full 48-bit word containing 6 characters)
    ASX ptr        ; A >>= (offset_enc * 8) (hardware right-shifts target byte to lowest 8 bits!)
    AAX =0377      ; A &= 0xFF (mask off upper bits)
```
This extracts any character from any position in $O(1)$ time without branches, loops, or lookup tables.

#### 2. Pointer Increment (`a++`) via Helper
Incrementing a byte pointer calls the runtime helper **`b/pinc`** (`c-compiler/libc/besm6/madlen/b_pinc.madlen`). It decrements `offset_enc` by 1 (stepping from Byte #0 toward Byte #5). When stepping past Byte #5 (`offset_enc` wraps from `0` to `5`), it increments the 15-bit word address by 1 and resets the offset to Byte #0 of the next word:

```madlen
    b/pinc: ,name,
            ,atx, p
            ,aax, =77777        . A = word address (bits 15-1)
            ,atx, w
            ,xta, p
            ,asn, 64+44         . offset_enc -> bits 3-1
            ,aax, =7            . offset_enc (0..5)
            ,uza, wrap          . offset_enc == 0 -> wrap to 5, bump word address
            ,arx, =7            . offset_enc - 1 (cyclic add mod 8)
            ...
```

#### 3. Byte Store (`*a = val`) via Helper
For modifying strings, the **`b/stb`** helper (`c-compiler/libc/besm6/madlen/b_stb.madlen`) performs an atomic read-modify-write on the containing word using offset-indexed mask and shift tables.

---

## 4. Architecture Comparison

To put the BESM-6 C data model in perspective, compare it against modern byte-addressed architectures:

| Property | BESM-6 | x86_64 (Linux) | AArch64 (ARM64) | RISC-V 32 | AVR (8-bit) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Addressable Unit** | **48-bit word** | 8-bit byte | 8-bit byte | 8-bit byte | 8-bit byte |
| **Hardware Byte Access** | **No** (Word only) | Yes (`MOV` byte) | Yes (`LDRB`/`STRB`) | Yes (`LB`/`SB`) | Yes (`LD`/`ST`) |
| **`CHAR_BIT`** | **8** | 8 | 8 | 8 | 8 |
| **`sizeof(char)`** | **1** (1 byte*) | 1 (1 byte) | 1 (1 byte) | 1 (1 byte) | 1 (1 byte) |
| **`sizeof(int)`** | **6** (1 word) | 4 (4 bytes) | 4 (4 bytes) | 4 (4 bytes) | 2 (2 bytes) |
| **`sizeof(long)`** | **6** (1 word) | 8 (8 bytes) | 8 (8 bytes) | 4 (4 bytes) | 4 (4 bytes) |
| **`sizeof(void*)`** | **6** (1 word) | 8 (8 bytes) | 8 (8 bytes) | 4 (4 bytes) | 2 (2 bytes) |
| **Pointer Layouts** | **2** (Regular & Fat) | 1 (Uniform virtual) | 1 (Uniform virtual) | 1 (Uniform virtual)| 1 (Uniform physical)|

*\*Note: In C, `sizeof` is measured in char-units (8-bit bytes). Because 6 char-units fit in a 48-bit word, any 1-word type (`int`, `long`, `void*`) has size 6. A standalone scalar `char` variable physically consumes 1 full word in RAM (wasting 40 bits), but its C `sizeof` is 1.*

---

## 5. References & Source Files

If you are exploring the codebases or adapting this design for other word-oriented or retro architectures, refer to these primary documentation and source files:

1. **Data Representation**: [`c-compiler/docs/Besm6_Data_Representation.md`](https://github.com/besm6/c-compiler/blob/master/docs/Besm6_Data_Representation.md) — Exhaustive breakdown of scalar bit layouts, integer ranges, and fat pointers.
2. **Runtime Helper Library**: [`c-compiler/docs/Besm6_Runtime_Library.md`](https://github.com/besm6/c-compiler/blob/master/docs/Besm6_Runtime_Library.md) — ABI contracts, register conventions, and Madlen assembly routines.
3. **C Calling Convention**: [`c-compiler/docs/Besm6_Calling_Conventions.md`](https://github.com/besm6/c-compiler/blob/master/docs/Besm6_Calling_Conventions.md) — Stack frame layout, `b/save`, and `b/ret`.
4. **String Library Implementation**: [`c-compiler/libc/besm6/strcmp.c`](https://github.com/besm6/c-compiler/blob/master/libc/besm6/strcmp.c) and [`strncmp.c`](https://github.com/besm6/c-compiler/blob/master/libc/besm6/strncmp.c).
5. **Assembly Helpers**: [`c-compiler/libc/besm6/madlen/b_pinc.madlen`](https://github.com/besm6/c-compiler/blob/master/libc/besm6/madlen/b_pinc.madlen) (`char*++`) and [`b_stb.madlen`](https://github.com/besm6/c-compiler/blob/master/libc/besm6/madlen/b_stb.madlen) (`*char = val`).
6. **Unix Version 7 Port**: [`v7besm/README.md`](https://github.com/besm6/v7besm) — Status of the kernel, toolchain, and simulator integration.
7. **BESM-6 Hardware Simulator**: [`simh/BESM6/README.md`](https://github.com/besm6/simh/tree/master/BESM6) — Guide to running DISPAK, attaching peripherals, and using the graphical front panel.
