# sc_bit - Single-Bit Class (Deprecated)

## Overview

`sc_bit` is the most basic bit type in SystemC, representing a value of either 0 or 1. **This class has been deprecated** -- it is recommended to use C++'s native `bool` type directly. Construction triggers a deprecation warning.

**Source file:** `sc_bit.h` + `sc_bit.cpp`

## Everyday Analogy

`sc_bit` is like the simplest light switch -- only two states: "on" and "off". But now C++ already has `bool` (also on/off), so there is no need for an extra wrapper. It is like buying a special "switch adapter" to control a lamp that already has a switch.

## Key Concepts

### Why Is It Deprecated?

In the early days of SystemC design, C++'s `bool` type was not yet standardized enough, so `sc_bit` was needed to ensure cross-platform consistency. Now that the C++ standard has matured `bool`, `sc_bit` has become redundant.

### Deprecation Mechanism

Each time an `sc_bit` is constructed, `sc_deprecated_sc_bit()` is called, which uses a static variable to ensure the warning is printed only once:

```cpp
sc_bit my_bit(1);
// First time: prints "sc_bit is deprecated, use bool instead"
// Subsequent times: no warning
```

## Class Interface

### Constructors

```cpp
sc_bit();                // default: false
sc_bit(bool a);          // from bool
sc_bit(char a);          // '0' or '1'
sc_bit(int a);           // 0 or 1
sc_bit(const sc_logic& a); // from sc_logic (must be 0 or 1)
```

All constructors are `explicit` to avoid unexpected implicit conversions.

### Value Conversion

```cpp
bool to_bool() const;    // convert to bool
char to_char() const;    // convert to '0' or '1'
```

### Bitwise Operations

```cpp
sc_bit& operator &= (const sc_bit& b);  // AND
sc_bit& operator |= (const sc_bit& b);  // OR
sc_bit& operator ^= (const sc_bit& b);  // XOR
const sc_bit operator ~ () const;        // NOT
```

Bitwise operations are internally just boolean operations: AND maps to `&&`, OR maps to `||`, XOR maps to `!=`.

### I/O

```cpp
void print(std::ostream& os) const;
void scan(std::istream& is);
friend std::ostream& operator<<(std::ostream&, const sc_bit&);
friend std::istream& operator>>(std::istream&, sc_bit&);
```

## Internal Implementation

- Internally stored as a `bool m_val`
- The `to_value()` family of static methods validates input values
- Invalid values (e.g., `char` not '0' or '1') call `invalid_value()`, which triggers `SC_REPORT_ERROR` and calls `sc_abort()`

## Error Handling

| Scenario | Error |
|----------|-------|
| `sc_bit('2')` | `SC_ID_VALUE_NOT_VALID_` |
| `sc_bit(5)` | `SC_ID_VALUE_NOT_VALID_` |

## Design Rationale / RTL Background

In RTL (Register Transfer Level) design, a single bit is the most fundamental signal unit. Verilog's `reg` and VHDL's `std_logic` both have corresponding single-bit types. `sc_bit` was originally designed for VSIA (Virtual Socket Interface Alliance) compatibility, but as the C++ standard matured, `bool` became sufficient.

## Related Files

- [sc_logic.md](sc_logic.md) - Four-valued logic type, the advanced version of `sc_bit`
- [sc_bit_ids.md](sc_bit_ids.md) - Error message ID definitions
- Source: `ref/systemc/src/sysc/datatypes/bit/sc_bit.h`
- Source: `ref/systemc/src/sysc/datatypes/bit/sc_bit.cpp`
