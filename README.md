# IOCLA CHEATSHEET

## 1. PRINTF formats
### a. The Standard Integer Cheat Sheet
| Size | Assembly Register | C Type (Signed / Unsigned) | Signed (`int`) | Unsigned | Hexadecimal (Lower / Upper) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **8-bit** | `al`, `bl`, `cl`, `dl` | `char` / `unsigned char` | `%hhd` | `%hhu` | `%hhx` / `%hhX` |
| **16-bit** | `ax`, `bx`, `cx`, `dx` | `short` / `unsigned short` | `%hd` | `%hu` | `%hx` / `%hX` |
| **32-bit** | `eax`, `ebx`, `ecx` | `int` / `unsigned int` | `%d` (or `%i`) | `%u` | `%x` / `%X` |
| **64-bit** | `rax`, `rbx`, `rcx` | `long` / `unsigned long` | `%ld` | `%lu` | `%lx` / `%lX` |

---

### b. Pointers, Text, and Floats
| Data Type | Description | Format Specifier | Example Output |
| :--- | :--- | :--- | :--- |
| **Memory Address** | Any Pointer (`void *`, `int *`, etc.) | `%p` | `0x7ffeb5b9a4c0` |
| **Character** | Single `char` (8-bit) | `%c` | `A` |
| **String** | Array of chars / `char *` (Null-terminated) | `%s` | `Hello World` |
| **Floating Point**| Single precision (`float` - 32-bit) | `%f` | `3.141593` |
| **Floating Point**| Double precision (`double` - 64-bit) | `%lf` | `3.1415926535` |

---

## Assembly Arithmetic
### a. Addition and Subtraction
The CPU uses the exact same instructions for both signed and unsigned numbers.
| Operation | Instruction | Example | Description |
| :--- | :--- | :--- | :--- |
| **Add** | `add` | `add rax, rbx` | `rax = rax + rbx` |
| **Add with Carry** | `adc` | `adc rax, rbx` | `rax = rax + rbx + CF` (Carry Flag) |
| **Increment** | `inc` | `inc rax` | `rax = rax + 1` (Faster than add 1) |
| **Subtract** | `sub` | `sub rax, rbx` | `rax = rax - rbx` |
| **Sub with Borrow** | `sbb` | `sbb rax, rbx` | `rax = rax - rbx - CF` |
| **Decrement** | `dec` | `dec rax` | `rax = rax - 1` |
| **Negate (Multiply by -1)**| `neg` | `neg rax` | Flips all bits and adds 1 (Two's comp) |

---

### b. Multiplication (Size Doubles)
When you multiply two numbers, the result can require twice as many bits. The CPU automatically splits the massive result across two registers: `rdx` (High bits) and `rax` (Low bits).
| Type | Instruction | Example | Implicit Math (Assuming 64-bit) |
| :--- | :--- | :--- | :--- |
| **Unsigned** | `mul` | `mul rbx` | `rdx:rax = rax * rbx` |
| **Signed** | `imul` | `imul rbx` | `rdx:rax = rax * rbx` |
| **Signed (Modern)**| `imul` | `imul rax, rbx`| `rax = rax * rbx` (Discards high bits, highly common in C compilers) |

---

### c. Division (Size Halves)
Division requires you to load a dividend twice as large as your divisor across two registers before you call the instruction. 
| Type | Instruction | Divisor Size | Implicit Dividend | Quotient | Remainder |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Unsigned** | `div` | 8-bit (`bl`) | `ax` (16-bit) | `al` | `ah` |
| **Signed** | `idiv`| 8-bit (`bl`) | `ax` (16-bit) | `al` | `ah` |
| **Unsigned** | `div` | 32-bit (`ebx`) | `edx:eax` (64-bit) | `eax` | `edx` |
| **Signed** | `idiv`| 32-bit (`ebx`) | `edx:eax` (64-bit) | `eax` | `edx` |

---

### d. Sign Extension (CRITICAL for `idiv`)
Before executing a signed division (`idiv`), you **must** stretch the sign bit of `rax` all the way across `rdx`. If `rax` is negative, `rdx` must be filled with `1`s. If positive, `0`s. 
| Original Register | Setup Instruction | Fills this Register | Purpose |
| :--- | :--- | :--- | :--- |
| `al` (8-bit) | `cbw` (Convert Byte to Word) | `ah` | Prep for 8-bit `idiv` |
| `ax` (16-bit) | `cwd` (Convert Word to Double) | `dx` | Prep for 16-bit `idiv` |
| `eax` (32-bit)| `cdq` (Convert Double to Quad)| `edx` | Prep for 32-bit `idiv` |
| `rax` (64-bit)| `cqo` (Convert Quad to Octo) | `rdx` | Prep for 64-bit `idiv` |

Example1:
```Assembly
; ---------------------------------------------------------
; Goal: Calculate -50 / 3 using signed 32-bit division
; ---------------------------------------------------------

section .text
global _start

_start:
    ; 1. Load the dividend into eax
    mov eax, -50       ; eax now holds 0xFFFFFFCE (-50 in Two's Complement)

    ; 2. Load the divisor into any other register
    mov ebx, 3         ; ebx now holds 0x00000003

    ; 3. CRITICAL STEP: Sign Extension
    cdq                ; "Convert Double to Quad"
                       ; Because the highest bit of eax is a 1 (it's negative),
                       ; cdq automatically fills edx entirely with 1s.
                       ; edx becomes 0xFFFFFFFF.
                       ; (If eax were positive, cdq would fill edx with 0s).
                       ; Now edx:eax perfectly represents a 64-bit -50!

    ; 4. Execute the division
    idiv ebx           ; CPU divides the 64-bit edx:eax by the 32-bit ebx
                       ; Result:
                       ; eax (Quotient)  = -16 (0xFFFFFFF0)
                       ; edx (Remainder) = -2  (0xFFFFFFFE)
```

Example2:
```Assembly
; ---------------------------------------------------------
; Goal: Calculate -50 / 3 using signed 64-bit division
; ---------------------------------------------------------

section .text
global _start

_start:
    ; 1. Load the 64-bit dividend into rax
    mov rax, -50       ; rax now holds 0xFFFFFFFFFFFFFFCE (-50)

    ; 2. Load the 64-bit divisor into another register
    mov rbx, 3         ; rbx now holds 0x0000000000000003

    ; 3. CRITICAL STEP: Sign Extension to 128-bit!
    cqo                ; "Convert Quad to Octo"
                       ; The CPU looks at the highest bit of rax (bit 63). 
                       ; Because it is a 1 (indicating a negative number),
                       ; cqo fills the entire 64-bit rdx register with 1s.
                       ; rdx becomes 0xFFFFFFFFFFFFFFFF.
                       ; rdx:rax is now a massive 128-bit mathematically 
                       ; perfect representation of -50!

    ; 4. Execute the division
    idiv rbx           ; CPU divides the 128-bit rdx:rax by the 64-bit rbx
                       ; Result:
                       ; rax (Quotient)  = -16 (0xFFFFFFFFFFFFFFF0)
                       ; rdx (Remainder) = -2  (0xFFFFFFFFFFFFFFFE)
```

# x86-64 Status Flags

The CPU automatically updates these flags in the `RFLAGS` register after almost every arithmetic (`add`, `sub`) or logical (`and`, `or`, `xor`) instruction.

| Flag Name | Abbreviation | When does it get set to `1`? | Assembly Example |
| :--- | :--- | :--- | :--- |
| **Zero Flag** | **ZF** | The result of an operation is exactly zero. (Highly used for checking equality). | `sub rax, rax` <br>*(Result is 0, ZF = 1)* |
| **Sign Flag** | **SF** | The highest bit (Most Significant Bit) of the result is a `1`. In Two's Complement, this means the result is negative. | `mov rax, 3` <br>`sub rax, 5` <br>*(Result is -2, SF = 1)* |
| **Carry Flag** | **CF** | An **unsigned** operation resulted in a value too large (or too small) to fit in the register. (Think of it as a carry-out or a borrow). | `mov al, 255` <br>`add al, 1` <br>*(al rolls over to 0, CF = 1)* |
| **Overflow Flag**| **OF** | A **signed** operation resulted in a value that crossed the positive/negative boundary incorrectly. (e.g., adding two positive numbers yields a negative result). | `mov al, 127` <br>`add al, 1` <br>*(Max signed 8-bit is 127. Result rolls to -128, OF = 1)* |

---

### How Comparison (`cmp`) Uses Flags
The `cmp` instruction is the most common way to trigger flags before a jump. 
**Secret:** `cmp rax, rbx` is exactly the same as `sub rax, rbx`, except it *throws away the math result* and only updates the flags!

* **If `rax == rbx`:** The hidden subtraction equals 0. **ZF** becomes `1`. (Use `je` - Jump if Equal).
* **If `rax < rbx` (Unsigned):** The hidden subtraction requires a borrow. **CF** becomes `1`. (Use `jb` - Jump if Below).
* **If `rax < rbx` (Signed):** The hidden subtraction results in a negative number. **SF** becomes `1` (usually). (Use `jl` - Jump if Less).
