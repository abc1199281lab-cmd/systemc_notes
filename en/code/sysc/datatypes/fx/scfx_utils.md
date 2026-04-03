# scfx_utils.h / .cpp -- Fixed-Point Utility Functions

## Overview

`scfx_utils.h` provides various **utility functions** needed for fixed-point string parsing and formatting. These functions handle radix prefix parsing, bit searching, digit character validation, and other low-level operations.

## Everyday Analogy

This file is like a "toolbox" containing various screwdrivers, wrenches, and measuring tools. Each tool is simple on its own, but they are indispensable components for assembling larger systems (string conversion, numeric display).

## Bit Search Functions

### `scfx_find_msb()` -- Find Most Significant Bit

```cpp
int scfx_find_msb(unsigned long x);
```

Uses binary search to find the highest non-zero bit position in `x`. For example, `scfx_find_msb(0b10100)` returns 4.

### `scfx_find_lsb()` -- Find Least Significant Bit

```cpp
int scfx_find_lsb(unsigned long x);
```

Finds the lowest non-zero bit position in `x`. For example, `scfx_find_lsb(0b10100)` returns 2.

## String Parsing Functions

### `scfx_parse_sign()` -- Parse Sign

```cpp
int scfx_parse_sign(const char*& s, bool& sign_char);
```

Reads `+` or `-` from the beginning of the string, returns 1 or -1, and advances the pointer.

### `scfx_parse_prefix()` -- Parse Radix Prefix

```cpp
sc_numrep scfx_parse_prefix(const char*& s);
```

Parses the string prefix to determine the radix:

| Prefix | Radix | Return Value |
|--------|-------|--------------|
| `0b` | Binary | `SC_BIN` |
| `0bus` | Binary unsigned | `SC_BIN_US` |
| `0bsm` | Binary sign-magnitude | `SC_BIN_SM` |
| `0o` | Octal | `SC_OCT` |
| `0x` | Hexadecimal | `SC_HEX` |
| `0d` | Decimal | `SC_DEC` |
| `0csd` | CSD (Canonical Signed Digit) | `SC_CSD` |
| No prefix | Decimal | `SC_DEC` |

### `scfx_parse_base()` -- Parse Base

```cpp
int scfx_parse_base(const char*& s);
```

Similar to `scfx_parse_prefix()`, but returns the numeric base (2, 8, 10, 16).

### Character Validation Functions

```cpp
bool scfx_is_digit(char c, sc_numrep numrep);  // is valid digit?
int scfx_to_digit(char c, sc_numrep numrep);   // char to numeric value
bool scfx_is_nan(const char* s);                // is "NaN"?
bool scfx_is_inf(const char* s);                // is "Inf" or "Infinity"?
bool scfx_exp_start(const char* s);             // starts with "e+" or "e-"?
```

## String Output Functions

### `scfx_print_nan()` / `scfx_print_inf()`

```cpp
void scfx_print_nan(scfx_string& s);
void scfx_print_inf(scfx_string& s, bool negative);
```

### `scfx_print_prefix()` -- Print Radix Prefix

```cpp
void scfx_print_prefix(scfx_string& s, sc_numrep numrep);
```

### `scfx_print_exp()` -- Print Exponent

```cpp
void scfx_print_exp(scfx_string& s, int exp);
```

Format like `"e+10"` or `"e-3"`, skipping leading zeros.

## CSD Conversion

```cpp
void scfx_tc2csd(scfx_string&, int);  // two's complement to CSD
void scfx_csd2tc(scfx_string&);       // CSD to two's complement
```

CSD (Canonical Signed Digit) is a numeric representation where each digit can be 0, 1, or -1, guaranteeing that adjacent digits are never both non-zero. This is used in hardware multiplier design to reduce the number of adders.

## .cpp File

`scfx_utils.cpp` primarily implements the CSD conversion functions `scfx_tc2csd()` and `scfx_csd2tc()`.

## Related Files

- `scfx_string.h` -- `scfx_string` class
- `scfx_rep.h` / `scfx_rep.cpp` -- Uses these utilities for string conversion
- `sc_fxdefs.h` -- `sc_numrep` enum definition
- `scfx_params.h` -- Included by this file
