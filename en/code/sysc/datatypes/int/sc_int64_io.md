# sc_int64_io.cpp — 64-Bit Integer I/O Support

## Overview

`sc_int64_io.cpp` provides input/output support functions for 64-bit integers, specifically to address compatibility issues with the Microsoft Visual C++ compiler (MSVC) when handling the `<<` operator for `uint64` and `int64` types.

**Source file:**
- `ref/systemc/src/sysc/datatypes/int/sc_int64_io.cpp`

## Everyday Analogy

This is like a translation service. Some old-fashioned printers (the MSVC compiler) do not recognize the new number format (`uint64`), so a translator is needed to convert the numbers into a format the printer understands (character strings) before printing.

## Core Functionality

### write_uint64

```cpp
static void write_uint64(::std::ostream& os, uint64 val, int sign);
```

Manually converts a `uint64` value to a string and writes it to the output stream. Supports:
- **Octal** format
- **Hexadecimal** format
- **Decimal** format
- Prefix display (`0x`, `0`)
- Positive sign display (`+`)

### Conditional Compilation

```cpp
#if defined( _MSC_VER )
// ... MSVC-specific implementations
#endif
```

The entire implementation is wrapped by the `_MSC_VER` macro and is only compiled under the MSVC compiler. Other compilers (GCC, Clang) already correctly support `uint64` I/O in their standard libraries.

## Other Includes

This file also indirectly triggers the expansion of some important inline functions:

```cpp
#include "sysc/datatypes/int/sc_signed_ops.h"
#include "sysc/datatypes/int/sc_signed_inlines.h"
#include "sysc/datatypes/int/sc_unsigned_inlines.h"
```

These includes ensure that functions requiring complete definitions from multiple header files are correctly instantiated.

## Related Files

- [sc_int_base.md](sc_int_base.md) — Class that uses I/O functionality
- [sc_uint_base.md](sc_uint_base.md) — Class that uses I/O functionality
- [sc_signed.md](sc_signed.md) — Inline functions expanded here
- [sc_unsigned.md](sc_unsigned.md) — Inline functions expanded here
