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


# x86-64 Signed vs. Unsigned Operations

### a. The "Sign-Agnostic" Instructions (Works for BOTH)
Because x86 uses Two's Complement binary, the hardware uses the exact same bit-flipping logic for these operations regardless of the sign.

* **Basic Math:** `add`, `sub`, `inc`, `dec`
* **Bitwise Logic:** `and`, `or`, `xor`, `not`, `test`
* **Bit Shift Left:** `shl` and `sal` (Shift Arithmetic Left). *Fun fact: In x86, these are literally the exact same instruction under the hood!*
* **Equality Jumps:** `je` (Jump Equal), `jne` (Jump Not Equal), `jz` (Jump Zero), `jnz` (Jump Not Zero)

---

### b. The Great Divide (Sign-Aware Instructions)
When an operation needs to change the physical size of a number, divide it, or compare which one is larger, you **must** choose the correct instruction based on your C data type (`int` vs. `unsigned int`).

| Operation Category | Signed Instruction (e.g., `int`) | Unsigned Instruction (e.g., `unsigned`) |
| :--- | :--- | :--- |
| **Multiplication** | `imul` (Integer Multiply) | `mul` (Unsigned Multiply) |
| **Division** | `idiv` (Integer Divide) | `div` (Unsigned Divide) |
| **Sign Extension (Div Prep)** | `cbw`, `cwd`, `cdq`, `cqo` <br>*(Fills upper register with Sign Bit)* | `xor edx, edx` / `xor rdx, rdx`<br>*(Manually clear upper register to 0s)* |
| **Widening (Copying sizes)** | `movsx` (Move with Sign Extend)<br>*(Copies the sign bit to fill new space)* | `movzx` (Move with Zero Extend)<br>*(Fills new space strictly with 0s)* |
| **Bit Shift Right** | `sar` (Shift Arithmetic Right)<br>*(Drags the sign bit along. Preserves negatives!)* | `shr` (Shift Right)<br>*(Pulls in fresh 0s from the left)* |

---

### c. The Comparison Jumps (`cmp` results)
If you use `cmp rax, rbx`, the CPU sets the flags. You must use the correct jump instruction so the CPU looks at the correct flags!

| Mathematical Question | Signed Jump (Looks at SF / OF) | Unsigned Jump (Looks at CF) |
| :--- | :--- | :--- |
| **Is A > B?** | `jg` (Jump Greater) | `ja` (Jump Above) |
| **Is A >= B?** | `jge` (Jump Greater or Equal) | `jae` (Jump Above or Equal) |
| **Is A < B?** | `jl` (Jump Less) | `jb` (Jump Below) |
| **Is A <= B?** | `jle` (Jump Less or Equal) | `jbe` (Jump Below or Equal) |

**The Golden Rule of Jumps:** * Use **Greater/Less** (`jg`, `jl`) for **Signed** numbers. 
* Use **Above/Below** (`ja`, `jb`) for **Unsigned** numbers (like memory addresses or sizes).

## The CF vs. OF

### The Golden Rule
* **Carry Flag (CF)** is strictly for **Unsigned** arithmetic.
* **Overflow Flag (OF)** is strictly for **Signed** arithmetic.

---

### i. High-Level Comparison

| Feature | Carry Flag (CF) | Overflow Flag (OF) |
| :--- | :--- | :--- |
| **Domain** | **Unsigned** numbers (e.g., `unsigned int`, memory addresses) | **Signed** numbers (e.g., `int`, Two's Complement) |
| **The Meaning** | The physical register ran out of space (a carry-out or borrow occurred). | The mathematical sign of the result makes no logical sense. |
| **Hardware Trigger** | The Most Significant Bit (MSB) pushed a `1` off the edge of the register. | The carry *into* the MSB does not match the carry *out* of the MSB. |
| **Relevant Jumps** | `ja` (Above), `jb` (Below), `jc` (Jump Carry) | `jg` (Greater), `jl` (Less), `jo` (Jump Overflow) |

---

### ii. Concrete Examples (Using the 8-bit `al` register)
* **8-bit Unsigned Range:** 0 to 255
* **8-bit Signed Range:** -128 to +127

#### Scenario A: The Carry Flag Triggers (CF = 1, OF = 0)
**The Code:**
```nasm
mov al, 255   ; Unsigned: 255. Signed: -1
add al, 1     ; Add 1 to al
```
* **Unsigned View:** `255 + 1 = 256`. 256 requires 9 bits and cannot fit in an 8-bit register. The CPU drops a bit off the edge. **CF is set to 1.**
* **Signed View:** `-1 + 1 = 0`. This is perfectly valid math, and `0` fits comfortably inside the signed limit. **OF is set to 0.**

#### Scenario B: The Overflow Flag Triggers (CF = 0, OF = 1)
**The Code:**
```nasm
mov al, 127   ; Unsigned: 127. Signed: +127
add al, 1     ; Add 1 to al
```
* **Unsigned View:** `127 + 1 = 128`. 128 easily fits inside the 255 max limit. No extra bits fell off the edge. **CF is set to 0.**
* **Signed View:** `+127 + 1 = +128`. The absolute maximum positive number an 8-bit signed register can hold is +127. Adding 1 causes the sign bit to accidentally flip to a `1` (negative). The CPU sees *Positive + Positive = Negative* and panics. **OF is set to 1.**

#### Scenario C: Both Flags Trigger (CF = 1, OF = 1)
**The Code:**
```nasm
mov al, 128   ; Unsigned: 128. Signed: -128
add al, 128   ; Add 128 to al
```
* **Unsigned View:** `128 + 128 = 256`. This exceeds the 255 limit, dropping a bit off the edge. **CF is set to 1.**
* **Signed View:** `-128 + -128 = -256`. The absolute minimum number is -128. Adding two negative numbers causes the sign bit to accidentally flip to `0` (positive). The CPU sees *Negative + Negative = Positive* and panics. **OF is set to 1.**

## GDB

### a. The Visual Dashboard (TUI Mode)
If you only learn one thing about GDB, make it this. By default, GDB is a blind text prompt. You can turn on a graphical layout that shows your registers and assembly updating in real-time.
| Command | Shortcut | Description |
| :--- | :--- | :--- |
| `layout asm` | | Splits the screen to show your assembly code live. |
| `layout reg` | | Splits the screen again to show all CPU registers and Flags! |
| `focus cmd` | | Moves your keyboard cursor back to the command prompt. |
| *Refresh* | `Ctrl + L` | If the screen glitches (common when `printf` prints over the UI), this redrawns the dashboard perfectly. |

---

### b. Execution Control (Time Travel)

| Command | Shortcut | Description |
| :--- | :--- | :--- |
| `run` | `r` | Starts the program. Runs until it hits a breakpoint or crashes. |
| `start` | | Starts the program but automatically pauses at the very first instruction (usually `main`). |
| `continue` | `c` | Unpauses the program. Runs until the next breakpoint. |
| `stepi` | `si` | **Step Instruction:** Executes exactly *one* assembly instruction. If it's a `call`, it jumps *inside* the function. |
| `nexti` | `ni` | **Next Instruction:** Executes exactly *one* assembly instruction. If it's a `call`, it executes the whole function and pauses on the next line. |
| `quit` | `q` | Exits GDB. |

---

### c. Planting Breakpoints (Trapping the CPU)
Breakpoints tell the CPU: *"Run at full speed, but slam the brakes right before you execute this specific line."*

| Command | Example | Description |
| :--- | :--- | :--- |
| `break <label>` | `b main` or `b jump_incoming` | Sets a breakpoint at a named function or assembly label. |
| `break *<address>`| `b *0x40114a` | Sets a breakpoint at a specific raw memory address (Crucial for CTFs without labels!). |
| `info break` | `i b` | Lists all your active breakpoints and their ID numbers. |
| `delete <ID>` | `d 1` | Deletes breakpoint #1. |

---

### d. Interrogating the State (Memory, Registers, and Flags)
When the program is paused, you use these commands to figure out exactly what the CPU is thinking.

| Command | Example | Description |
| :--- | :--- | :--- |
| `info registers` | `i r` | Dumps the current value of all registers. |
| `print $<reg>` | `p $rax` or `p/x $rax`| Prints a specific register. Use `p/x` to print it in Hexadecimal. *(Note the `$` prefix!)* |
| `examine` | `x/x $rip` | The **Examine** command. Looks at raw memory. This example shows the hex byte exactly where `rip` is pointing. |
| `examine string` | `x/s $rdi` | Looks at the memory address inside `$rdi` and prints it as a text string. (Amazing for finding hidden passwords in CTFs). |

---

### e. Reading the RFLAGS Register (The `ZF` Mystery)

If you type `i r eflags` (Info Registers: EFLAGS), GDB will print something like this:
`eflags         0x246    [ PF ZF IF ]`

GDB is incredibly helpful here. Instead of making you read the raw hex (`0x246`), it puts the active flags inside brackets `[ ]`. 
* If you see **`ZF`** in the brackets, the Zero Flag is **1** (Set).
* If `ZF` is missing from the brackets, the Zero Flag is **0** (Cleared).
