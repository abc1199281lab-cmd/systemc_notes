# sc_string - String Utilities and Number Representation

## Overview

`sc_string.h` provides the number representation enumeration definition and I/O stream helper functions. Despite the file name being "sc_string", its current main contents are the `sc_numrep` enumeration and related I/O utilities, rather than a string class implementation.

**Source file**: `sysc/utils/sc_string.h` (header only)

## Analogy

When you want to tell someone a number, you can use different representations:
- Decimal: 255
- Hexadecimal: 0xFF
- Binary: 11111111
- Octal: 0377

`sc_numrep` enumerates all number representations supported by SystemC, letting the program know which representation to use when displaying a value.

## sc_numrep Enumeration

```cpp
enum sc_numrep {
    SC_NOBASE = 0,    // no base
    SC_BIN    = 2,    // binary
    SC_OCT    = 8,    // octal
    SC_DEC    = 10,   // decimal
    SC_HEX    = 16,   // hexadecimal
    SC_BIN_US,        // binary unsigned
    SC_BIN_SM,        // binary sign-magnitude
    SC_OCT_US,        // octal unsigned
    SC_OCT_SM,        // octal sign-magnitude
    SC_HEX_US,        // hexadecimal unsigned
    SC_HEX_SM,        // hexadecimal sign-magnitude
    SC_CSD            // Canonical Signed Digit
};
```

Note that the values of `SC_BIN`, `SC_OCT`, `SC_DEC`, `SC_HEX` are the corresponding base numbers (2, 8, 10, 16), making them convenient for direct use in calculations.

### US and SM Suffixes

- **US (Unsigned)**: Treats the bit pattern as an unsigned number
- **SM (Sign-Magnitude)**: The most significant bit is the sign bit, the rest represent the magnitude

## I/O Helper Functions

### sc_io_base -- Get the Stream's Base

```cpp
inline sc_numrep sc_io_base(systemc_ostream& stream, sc_numrep def_base);
```

Returns the corresponding `sc_numrep` based on the stream's format flags (`std::ios::basefield`):
- `std::ios::dec` -> `SC_DEC`
- `std::ios::hex` -> `SC_HEX`
- `std::ios::oct` -> `SC_OCT`
- Otherwise -> `def_base` (default value)

### sc_io_show_base -- Whether to Show Base Prefix

```cpp
inline bool sc_io_show_base(systemc_ostream& stream);
```

Returns whether the stream has the `std::ios::showbase` flag set (e.g., whether to display the `0x` prefix).

## Backward Compatibility

```cpp
#ifdef SC_USE_STD_STRING
typedef ::std::string sc_string;
#endif
```

If `SC_USE_STD_STRING` is defined, `sc_string` becomes an alias for `std::string`, for compatibility with legacy code.

## Namespace

Note that `sc_numrep` and related functions are defined in the `sc_dt` namespace (not `sc_core`), because they are related to data types.

## Related Files

- [sc_string_view.md](sc_string_view.md) -- Non-owning string reference
- [sc_iostream.md](sc_iostream.md) -- I/O stream header wrapper
