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

### c. Sign Extension (CRITICAL for `idiv`)
Before executing a signed division (`idiv`), you **must** stretch the sign bit of `rax` all the way across `rdx`. If `rax` is negative, `rdx` must be filled with `1`s. If positive, `0`s. 
| Original Register | Setup Instruction | Fills this Register | Purpose |
| :--- | :--- | :--- | :--- |
| `al` (8-bit) | `cbw` (Convert Byte to Word) | `ah` | Prep for 8-bit `idiv` |
| `ax` (16-bit) | `cwd` (Convert Word to Double) | `dx` | Prep for 16-bit `idiv` |
| `eax` (32-bit)| `cdq` (Convert Double to Quad)| `edx` | Prep for 32-bit `idiv` |
| `rax` (64-bit)| `cqo` (Convert Quad to Octo) | `rdx` | Prep for 64-bit `idiv` |
