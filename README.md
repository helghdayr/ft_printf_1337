# ft_printf

> A re-implementation of the C standard `printf` function — handling multiple format specifiers, variadic arguments, and formatted output to stdout, built entirely from scratch without any standard I/O functions.

---

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Compilation](#compilation)
- [Supported Conversions](#supported-conversions)
- [How It Works](#how-it-works)
  - [Variadic Functions](#variadic-functions)
  - [Format String Parsing](#format-string-parsing)
  - [Dispatch Logic](#dispatch-logic)
- [File by File](#file-by-file)
  - [ft_printf.c](#ft_printfc)
  - [print_char.c](#print_charc)
  - [print_str.c](#print_strc)
  - [print_int.c](#print_intc)
  - [print_uint.c](#print_uintc)
  - [print_hex.c](#print_hexc)
  - [print_adress.c](#print_adressc)
- [Return Value](#return-value)
- [Key Concepts](#key-concepts)
- [Mandatory Rules](#mandatory-rules)

---

## Overview

**ft_printf** is a 42/1337 School project. The goal is to recode the `printf` function from `<stdio.h>` — one of the most commonly used functions in C — handling a variable number of arguments and formatting them into a human-readable output string.

You are **not allowed** to use `printf` itself or any buffered I/O function. Output is done exclusively with `write`.

The result is a static library: **`libftprintf.a`**

---

## Project Structure

```
ft_printf/
├── Makefile
├── ft_printf.h
├── ft_printf.c        # entry point — parses format string, dispatches specifiers
├── print_char.c       # handles %c
├── print_str.c        # handles %s
├── print_int.c        # handles %d and %i
├── print_uint.c       # handles %u
├── print_hex.c        # handles %x and %X
└── print_adress.c     # handles %p
```

---

## Compilation

```bash
# Build the library
make

# Clean object files
make clean

# Clean everything including the library
make fclean

# Rebuild from scratch
make re
```

Link it into your project with:

```bash
cc main.c -L. -lftprintf -o program
# or if combined with libft:
cc main.c -L. -lftprintf -lft -o program
```

---

## Supported Conversions

| Specifier | Meaning | Example input | Example output |
|-----------|---------|---------------|----------------|
| `%c` | Single character | `'A'` | `A` |
| `%s` | String | `"hello"` | `hello` |
| `%d` | Signed decimal integer | `-42` | `-42` |
| `%i` | Signed decimal integer | `42` | `42` |
| `%u` | Unsigned decimal integer | `4294967295` | `4294967295` |
| `%x` | Hexadecimal integer (lowercase) | `255` | `ff` |
| `%X` | Hexadecimal integer (uppercase) | `255` | `FF` |
| `%p` | Pointer address | `&var` | `0x7ffd...` |
| `%%` | Literal percent sign | — | `%` |

---

## How It Works

### Variadic Functions

> **Concept:** Normal C functions have a fixed number of parameters. `printf`-style functions accept **a variable number of arguments** of **different types** — this is done with the `<stdarg.h>` macros.

```c
#include <stdarg.h>

int ft_printf(const char *format, ...)
{
    va_list args;

    va_start(args, format);   // initialize the list after the last fixed param
    // ... use va_arg(args, type) to extract each argument
    va_end(args);             // mandatory cleanup
}
```

| Macro | Purpose |
|-------|---------|
| `va_list args` | Declares the argument list object |
| `va_start(args, last)` | Points `args` to the first variadic argument (after `last`) |
| `va_arg(args, type)` | Extracts the next argument as the given type and advances the pointer |
| `va_end(args)` | Cleans up — must always be called before returning |

The caller and callee share no type information at runtime — it is entirely the programmer's responsibility to pass the right type for each specifier. Passing the wrong type causes **undefined behavior**.

---

### Format String Parsing

The format string is walked character by character. When a `%` is encountered, the next character determines which conversion to apply. Every other character is printed as-is.

```
ft_printf("Hello %s, you are %d years old!\n", "Ali", 20);
           │      │           │
           │      │           └── '%d' → print_int(args)
           │      └── '%s' → print_str(args)
           └── "Hello " → write directly to stdout
```

```c
while (*format)
{
    if (*format == '%')
    {
        format++;           // skip the '%'
        dispatch(*format, &args, &count);
    }
    else
        print_char_literal(*format, &count);
    format++;
}
```

---

### Dispatch Logic

After the `%` is detected, a single character identifies the conversion. A series of `if`/`else if` or a dispatch approach routes to the correct handler:

```c
if      (specifier == 'c')  print_char(va_arg(args, int), &count);
else if (specifier == 's')  print_str(va_arg(args, char *), &count);
else if (specifier == 'd'
      || specifier == 'i')  print_int(va_arg(args, int), &count);
else if (specifier == 'u')  print_uint(va_arg(args, unsigned int), &count);
else if (specifier == 'x')  print_hex(va_arg(args, unsigned int), 0, &count);
else if (specifier == 'X')  print_hex(va_arg(args, unsigned int), 1, &count);
else if (specifier == 'p')  print_adress(va_arg(args, void *), &count);
else if (specifier == '%')  write(1, "%", 1), count++;
```

Every handler writes its output directly with `write(1, ...)` and updates a running **character count** that is returned at the end.

---

## File by File

---

### `ft_printf.c`

**Role:** Entry point and format string parser.

- Declares `va_list` and calls `va_start`.
- Loops over every character of `format`.
- When it hits `%`, advances one character and calls the appropriate handler.
- Accumulates the total number of characters written in a counter.
- Calls `va_end` and returns the counter.

```c
int ft_printf(const char *format, ...);
```

---

### `print_char.c`

**Role:** Handles the `%c` specifier.

> **Concept:** `%c` prints a single character. The argument is passed as `int` (not `char`) because C promotes all integer types narrower than `int` to `int` when passed through variadic arguments (`va_arg` rule). The function casts it back to `unsigned char` before writing.

```c
int print_char(int c);
// writes 1 byte to stdout via write(1, &c, 1)
// returns 1 (number of characters written)
```

**Edge case:** `%c` with a null byte (`'\0'`) must still write the byte and count it — unlike strings, it does not terminate output.

---

### `print_str.c`

**Role:** Handles the `%s` specifier.

> **Concept:** A string in C is a `char *` pointing to a sequence of bytes terminated by `'\0'`. `print_str` writes each character using `write` until the null terminator is reached. It does **not** write the terminator itself.

```c
int print_str(char *str);
// returns the number of characters written
```

**Edge case:** If the pointer is `NULL`, the original `printf` prints `(null)`. Your implementation must handle this too:

```c
if (!str)
    str = "(null)";
```

---

### `print_int.c`

**Role:** Handles `%d` and `%i` — signed decimal integers.

> **Concept:** To print a number as a string of decimal digits, extract digits one by one using modulo and division. Since this naturally produces digits in reverse order (least significant first), the function either recurses or uses a buffer. Negative numbers require printing a `-` sign first, then processing the absolute value.

```c
int print_int(int n);
// returns number of characters written (including '-' if negative)
```

**Edge case — `INT_MIN` (-2147483648):**  
`INT_MIN` cannot be negated safely — `-INT_MIN` overflows. It must be handled as a special case or by casting to `long` before negating.

```c
// Recursive approach (clean and elegant):
void    print_int_recursive(long n, int *count)
{
    if (n < 0) { write(1, "-", 1); (*count)++; n = -n; }
    if (n >= 10) print_int_recursive(n / 10, count);
    char c = '0' + (n % 10);
    write(1, &c, 1);
    (*count)++;
}
```

---

### `print_uint.c`

**Role:** Handles `%u` — unsigned decimal integers.

> **Concept:** Identical logic to `print_int` but without any sign handling. The type is `unsigned int`, so values range from `0` to `4294967295`. The same recursive digit extraction applies, but there is no negative branch.

```c
int print_uint(unsigned int n);
// returns number of characters written
```

**Why a separate file?** Although the logic is similar to `print_int`, mixing signed and unsigned arithmetic is a common source of bugs. Keeping them separate avoids accidental signed/unsigned comparison warnings and makes the intent explicit.

---

### `print_hex.c`

**Role:** Handles `%x` (lowercase) and `%X` (uppercase) — hexadecimal integers.

> **Concept:** Hexadecimal (base 16) uses digits `0–9` and letters `a–f` (or `A–F`). The digit extraction is the same modulo/division loop as decimal, but with base 16 and an index into a character lookup table instead of `'0' + remainder`.

```c
int print_hex(unsigned int n, int uppercase);
// uppercase == 0 → uses "0123456789abcdef"
// uppercase == 1 → uses "0123456789ABCDEF"
// returns number of characters written
```

**Lookup table approach:**
```c
char *base_low = "0123456789abcdef";
char *base_up  = "0123456789ABCDEF";

// extract digit: base[n % 16]
// recurse:       print_hex(n / 16, uppercase)
```

**Edge case:** `n == 0` must print `"0"`, not an empty string.

---

### `print_adress.c`

**Role:** Handles `%p` — pointer addresses.

> **Concept:** A pointer is just a memory address — an unsigned integer wide enough to hold any address on the current platform (`void *`, typically 8 bytes on 64-bit systems). `%p` prints this address in **hexadecimal** prefixed with `0x`. The type to extract from `va_list` is `void *`, which is then cast to `unsigned long` (or `uintptr_t`) for digit extraction.

```c
int print_adress(void *ptr);
// prints "0x" followed by the address in lowercase hex
// returns number of characters written (including "0x")
```

**Why not reuse `print_hex` directly?**  
Pointers are wider than `unsigned int` on 64-bit systems (8 bytes vs 4 bytes). Using `unsigned int` would truncate the upper 32 bits of the address and produce wrong output. `print_adress` must use `unsigned long` or `uintptr_t` throughout.

**Edge case:** `NULL` pointer — `printf` prints `(nil)`. Your implementation should do the same:

```c
if (!ptr)
{
    write(1, "(nil)", 5);
    return (5);
}
```

---

## Return Value

`ft_printf` returns the **total number of characters written** to stdout, exactly like the original `printf`. This count is accumulated across all handler calls and returned at the end of the function.

```c
int count = 0;

// ... each handler adds its written count to `count` ...

return (count);
```

Return `-1` on write error (optional but good practice).

---

## Key Concepts

### `write` vs `printf`

All output is done with the `write` system call:

```c
write(int fd, const void *buf, size_t count);
```

- `fd = 1` → stdout
- `buf` → pointer to data to write
- `count` → number of bytes to write
- Returns the number of bytes actually written (or -1 on error)

`write` is **unbuffered** — each call goes directly to the OS. This is slower than `printf`'s internal buffering but simpler to reason about and exactly what the project requires.

### Base Conversion

All numeric printing reduces to the same algorithm — extract digits in a given base using modulo and division, then output them in the correct (reversed) order:

```
255 in base 16:
  255 % 16 = 15  → 'f'   (last digit)
  255 / 16 = 15
   15 % 16 = 15  → 'f'   (first digit)
   15 / 16 = 0   → stop

Result: "ff"  (read in reverse from recursion)
```

The same logic works for base 10 (`%d`, `%u`) and base 16 (`%x`, `%X`) — only the base value and the digit character table change.

### Character Type Promotion in Variadic Functions

When a value narrower than `int` is passed through `...`, C automatically **promotes** it:

| Passed type | Promoted to | Extract with |
|-------------|-------------|--------------|
| `char` | `int` | `va_arg(args, int)` |
| `short` | `int` | `va_arg(args, int)` |
| `float` | `double` | `va_arg(args, double)` |

Always use the promoted type in `va_arg` — using the original narrow type is **undefined behavior**.

---

## Mandatory Rules

- [x] No use of `printf`, `sprintf`, or any function from `<stdio.h>` (except `write`)
- [x] No global variables
- [x] All files compile with `cc -Wall -Wextra -Werror`
- [x] Makefile includes `all`, `clean`, `fclean`, `re` rules — no unnecessary relinking
- [x] `libftprintf.a` is not committed to the repository
- [x] `%%` correctly outputs a single `%` and counts as 1 character
- [x] `%s` with `NULL` prints `(null)`
- [x] `%p` with `NULL` prints `(nil)`
- [x] `INT_MIN` is handled correctly by `%d` / `%i`
- [x] Norminette compliant (1337 / 42 coding norm)

---

## Author

**Login:** `hael-ghd`  
**School:** 1337 / 42  
**Project:** ft_printf
