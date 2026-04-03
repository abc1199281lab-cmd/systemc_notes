# sc_unsigned — Arbitrary-Precision Unsigned Integer

## Overview

`sc_unsigned` is the core class for arbitrary-precision unsigned integers in SystemC, and also the base class of `sc_biguint<W>`. It is the sister class of `sc_signed`, sharing most of the implementation code, but all operations are performed in an unsigned manner.

An important characteristic: **an n-bit `sc_unsigned` is equivalent to an (n+1)-bit non-negative `sc_signed`**. This is because `sc_unsigned` needs an extra bit to represent "sign bit is 0" (positive).

**Source files:**
- `ref/systemc/src/sysc/datatypes/int/sc_unsigned.h`
- `ref/systemc/src/sysc/datatypes/int/sc_unsigned.cpp`
- `ref/systemc/src/sysc/datatypes/int/sc_unsigned_inlines.h`
- `ref/systemc/src/sysc/datatypes/int/sc_unsigned_friends.h`

## Everyday Analogy

`sc_unsigned` is like an "extra-large odometer" — it can have arbitrarily many digits but will never display a negative number. Need a 256-bit address space? No problem. Need a 1024-bit encryption key? That works too.

## Class Structure

```mermaid
classDiagram
    class sc_value_base {
        <<abstract>>
    }

    class sc_unsigned {
        -sc_digit* digit
        -int nbits
        -int ndigits
        -small_type sgn
        +sc_unsigned(int nb)
        +operator=(various)
        +operator[](int i) sc_unsigned_bitref
        +range(int hi, int lo) sc_unsigned_subref
        +to_uint() unsigned int
        +to_uint64() uint64
        +length() int
    }

    sc_value_base <|-- sc_unsigned
    sc_unsigned <|-- sc_biguint_W["sc_biguint&lt;W&gt;"]
```

## Core Concepts

### 1. Internal Representation

Identical to `sc_signed`, using sign-magnitude representation:

```cpp
small_type sgn;      // always SC_POS or SC_ZERO (never SC_NEG)
sc_digit*  digit;    // magnitude array
int        nbits;    // number of bits
int        ndigits;  // number of sc_digit elements
```

Because it is an unsigned integer, `sgn` is never `SC_NEG`.

### 2. Code Sharing with sc_signed

`sc_unsigned` and `sc_signed` share most of the implementation code. Macros are used in the source code to control differences:

```cpp
// In shared implementation files:
#ifdef IF_SC_SIGNED
    // signed-specific code
#else
    // unsigned-specific code
#endif
```

### 3. Special Behavior of Unary Operators

```cpp
sc_unsigned operator + (const sc_unsigned& u);  // returns sc_unsigned
sc_signed   operator - (const sc_unsigned& u);  // negation returns sc_signed!
sc_signed   operator ~ (const sc_unsigned& u);  // bitwise NOT returns sc_signed!
```

Note: negating or bitwise-inverting an `sc_unsigned` returns `sc_signed`, because negative numbers cannot be represented in an unsigned type.

### 4. Mixed Operation Semantics

| Operation | Result Type |
|-----------|-------------|
| `sc_unsigned + sc_unsigned` | `sc_unsigned` |
| `sc_unsigned + sc_signed` | `sc_signed` |
| `sc_unsigned - sc_unsigned` | `sc_signed` |
| `-sc_unsigned` | `sc_signed` |

### 5. Proxy Classes

Symmetric with `sc_signed`, four proxy classes are provided:
- `sc_unsigned_bitref_r` / `sc_unsigned_bitref`: bit selection
- `sc_unsigned_subref_r` / `sc_unsigned_subref`: range selection

### 6. File Responsibilities

| File | Responsibility |
|------|----------------|
| `sc_unsigned.h` | Class declaration and proxy classes |
| `sc_unsigned.cpp` | Core implementation |
| `sc_unsigned_inlines.h` | Deferred inline function definitions |
| `sc_unsigned_friends.h` | Friend operator declarations |

## Usage Examples

```cpp
sc_unsigned addr(256);  // 256-bit address
addr = "0x1234_5678_9ABC_DEF0_1234_5678_9ABC_DEF0";

sc_unsigned key(1024);  // 1024-bit encryption key

// Arithmetic
sc_unsigned a(128), b(128);
sc_unsigned sum = a + b;    // result may need 129 bits
sc_signed diff = a - b;     // difference can be negative -> sc_signed
```

## RTL Correspondence

```
// Verilog
reg [255:0] wide_addr;
wire [127:0] data_a, data_b;

// SystemC
sc_unsigned wide_addr(256);
sc_unsigned data_a(128), data_b(128);
```

## Related Files

- [sc_biguint.md](sc_biguint.md) — Template subclass `sc_biguint<W>`
- [sc_signed.md](sc_signed.md) — Signed version `sc_signed`
- [sc_nbutils.md](sc_nbutils.md) — Low-level vector operations
- [sc_nbdefs.md](sc_nbdefs.md) — Basic type definitions
