# IOCLA CHEATSHEET

## 1. PRINTF formats
### a. The Standard Integer Cheat Sheet
| Size | Assembly Register | C Type (Signed / Unsigned) | Signed (`int`) | Unsigned | Hexadecimal (Lower / Upper) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **8-bit** | `al`, `bl`, `cl`, `dl` | `char` / `unsigned char` | `%hhd` | `%hhu` | `%hhx` / `%hhX` |
| **16-bit** | `ax`, `bx`, `cx`, `dx` | `short` / `unsigned short` | `%hd` | `%hu` | `%hx` / `%hX` |
| **32-bit** | `eax`, `ebx`, `ecx` | `int` / `unsigned int` | `%d` (or `%i`) | `%u` | `%x` / `%X` |
| **64-bit** | `rax`, `rbx`, `rcx` | `long` / `unsigned long` | `%ld` | `%lu` | `%lx` / `%lX` |

### b. Pointers, Text, and Floats
| Data Type | Description | Format Specifier | Example Output |
| :--- | :--- | :--- | :--- |
| **Memory Address** | Any Pointer (`void *`, `int *`, etc.) | `%p` | `0x7ffeb5b9a4c0` |
| **Character** | Single `char` (8-bit) | `%c` | `A` |
| **String** | Array of chars / `char *` (Null-terminated) | `%s` | `Hello World` |
| **Floating Point**| Single precision (`float` - 32-bit) | `%f` | `3.141593` |
| **Floating Point**| Double precision (`double` - 64-bit) | `%lf` | `3.1415926535` |

---

```
