# SystemC Datatypes Module Analysis

> Deep dive into SystemC datatypes: Bit vectors, Logic values, Integers, Fixed-point
> 
> Date: 2026-02-08

---

## 1. Overview: Datatype Architecture

SystemC provides hardware-oriented datatypes that differ from standard C++ types:

| Category | Types | Use Case |
|:---------|:------|:---------|
| Bit/Logic | sc_bit, sc_logic, sc_bv, sc_lv | Binary/4-value logic |
| Integer | sc_int, sc_uint, sc_bigint, sc_biguint | Sized integers |
| Fixed-point | sc_fixed, sc_ufixed, sc_fix | DSP applications |

---

## 2. sc_bit vs sc_logic

**Files**: 
- `ref/systemc/src/sysc/datatypes/bit/sc_bit.h`
- `ref/systemc/src/sysc/datatypes/bit/sc_logic.h`

### sc_bit: Two-Value Type (0, 1)

```cpp
class SC_API sc_bit
{
    bool m_val;  // Simple boolean storage

public:
    sc_bit() : m_val( false ) {}
    sc_bit( bool a ) : m_val( a ) {}
    
    // Implicit conversion to bool
    operator bool () const { return m_val; }
    
    // Bitwise operators
    sc_bit& operator &= ( const sc_bit& b );
    sc_bit& operator |= ( const sc_bit& b );
    sc_bit& operator ^= ( const sc_bit& b );
};
```

**Deprecated**: SystemC recommends using C++ `bool` instead.

### sc_logic: Four-Value Type (0, 1, X, Z)

```cpp
enum sc_logic_value_t
{
    Log_0 = 0,  // Logic 0
    Log_1,      // Logic 1
    Log_Z,      // High impedance
    Log_X       // Unknown/undefined
};

class SC_API sc_logic
{
    sc_logic_value_t m_val;
    
    // Truth tables for operations
    static const sc_logic_value_t and_table[4][4];
    static const sc_logic_value_t or_table[4][4];
    static const sc_logic_value_t xor_table[4][4];
    static const sc_logic_value_t not_table[4];
    
public:
    sc_logic() : m_val( Log_X ) {}  // Default to unknown
    
    sc_logic& operator &= ( const sc_logic& b )
        { m_val = and_table[m_val][b.m_val]; return *this; }
    
    sc_logic& operator |= ( const sc_logic& b )
        { m_val = or_table[m_val][b.m_val]; return *this; }
};
```

### Truth Table Example (AND):

| AND | 0 | 1 | X | Z |
|:----|:--|:--|:--|:--|
| 0   | 0 | 0 | 0 | 0 |
| 1   | 0 | 1 | X | X |
| X   | 0 | X | X | X |
| Z   | 0 | X | X | X |

### RTL Mapping:

| Type | Values | Hardware Use |
|:-----|:-------|:-------------|
| bool/sc_bit | 0, 1 | Control signals, flags |
| sc_logic | 0, 1, X, Z | Data buses, tri-state |

X (unknown) occurs when:
- Uninitialized flip-flops
- Conflicting drivers on same wire
- Floating inputs

Z (high-impedance) is used for:
- Tri-state buses
- Disconnected outputs
- Bidirectional pins in input mode

---

## 3. sc_bv: Bit Vector

**File**: `ref/systemc/src/sysc/datatypes/bit/sc_bv.h`

```cpp
template <int W>
class sc_bv : public sc_bv_base
{
public:
    sc_bv() : sc_bv_base( W ) {}
    sc_bv( const char* a ) : sc_bv_base( W )
        { sc_bv_base::operator = ( a ); }
    
    // Assignment from various types
    sc_bv<W>& operator = ( const char* a );
    sc_bv<W>& operator = ( const bool* a );
    sc_bv<W>& operator = ( const sc_logic* a );
    sc_bv<W>& operator = ( unsigned long a );
    sc_bv<W>& operator = ( long a );
    // ... more operators
};
```

### Bit Vector Operations:

| Operation | Description | RTL Equivalent |
|:----------|:------------|:---------------|
| range(i, j) | Bit slice | `reg[7:0]` |
| operator[] | Single bit access | `wire[3]` |
| and_reduce() | AND all bits | `&bus` |
| or_reduce() | OR all bits | `\|bus` |
| xor_reduce() | XOR all bits | `^bus` |

### sc_lv: Logic Vector (sc_bv with X/Z support)

```cpp
template <int W>
class sc_lv : public sc_lv_base
{
    // Same interface as sc_bv but supports 0, 1, X, Z per bit
};
```

### RTL Knowledge: Why Different Types?

**In Verilog:**
```verilog
wire [31:0] data_bus;      // Can contain X, Z
reg  [7:0]  byte_reg;      // Only 0, 1 after initialization
```

- `sc_bv<W>` → Optimized for 2-value logic (faster simulation)
- `sc_lv<W>` → Full 4-value logic for accurate hardware modeling

---

## 4. Integer Types: sc_int and sc_bigint

**File**: `ref/systemc/src/sysc/datatypes/int/sc_int.h`

### sc_int<W>: Fixed-Width Integer (< 64 bits)

```cpp
template <int W>
class sc_int : public sc_int_base
{
    void assign( uint_type value )
    {
        m_val = ( value & (1ull << (W-1)) ) ?  
                value | (~UINT64_ZERO << (W-1)) :  // Sign extend
                value & ( ~UINT_ZERO >> (SC_INTWIDTH-W) );
    }

public:
    sc_int() : sc_int_base( W ) {}
    sc_int( int_type v ) : sc_int_base( W ) { assign(v); }
    
    sc_int<W>& operator = ( int_type v )
        { assign( v ); return *this; }
    
    // Arithmetic operators
    sc_int<W>& operator += ( int_type v );
    sc_int<W>& operator -= ( int_type v );
    sc_int<W>& operator *= ( int_type v );
    sc_int<W>& operator /= ( int_type v );
    
    // Bitwise operators
    sc_int<W>& operator &= ( int_type v );
    sc_int<W>& operator |= ( int_type v );
    sc_int<W>& operator <<= ( int_type v );
    sc_int<W>& operator >>= ( int_type v );
};
```

### Key Features:

1. **Fixed Width**: Width specified at compile time via template parameter
2. **Automatic Sign Extension**: When assigning narrower to wider
3. **Overflow Handling**: Wraps around like hardware (not C++ undefined behavior)
4. **Bit Select**: Can access individual bits like hardware buses

### sc_bigint<W>: Arbitrary Precision (> 64 bits)

For integers wider than 64 bits:

```cpp
template <int W>
class sc_bigint : public sc_signed
{
    // Arbitrary precision using dynamic storage
};
```

| Type | Width | Use Case |
|:-----|:------|:---------|
| `int` | 32-bit | General software |
| `sc_int<8>` | 8-bit | Byte operations, memory addresses |
| `sc_int<16>` | 16-bit | DSP coefficients, short integers |
| `sc_int<32>` | 32-bit | Standard integer operations |
| `sc_int<64>` | 64-bit | Wide counters, addresses |
| `sc_bigint<128>` | 128-bit | Cryptography, wide accumulators |

---

## 5. Fixed-Point Types

**File**: `ref/systemc/src/sysc/datatypes/fx/sc_fix.h`

### Fixed-Point Representation

Fixed-point uses a fixed number of bits for integer and fractional parts:

```
Number format: | Sign | Integer Part | Fractional Part |
                 1 bit    WL-iwl bits      iwl bits
                 
Where: WL = total word length, iwl = integer word length
```

### sc_fixed: Templated Fixed-Point

```cpp
template <int WL, int IWL, 
          sc_q_mode QM = SC_TRN,
          sc_o_mode OM = SC_WRAP,
          int N = 0>
class sc_fixed : public sc_fix
{
    // WL = total word length
    // IWL = integer word length
    // QM = quantization mode (how to round)
    // OM = overflow mode (how to handle overflow)
    // N = number of saturation bits
};
```

### Quantization Modes:

| Mode | Description |
|:-----|:------------|
| SC_TRN | Truncate (discard LSBs) |
| SC_RND | Round to nearest |
| SC_RND_ZERO | Round to zero |
| SC_RND_INF | Round to infinity |
| SC_RND_MIN_INF | Round to minus infinity |

### Overflow Modes:

| Mode | Description |
|:-----|:------------|
| SC_WRAP | Wrap around (modulo) |
| SC_SAT | Saturate to max/min |
| SC_SAT_ZERO | Saturate to zero |
| SC_SAT_SYM | Symmetric saturation |

### RTL Knowledge: Fixed-Point in Hardware

Fixed-point is common in DSP chips because:

1. **Area**: Fixed-point multipliers are smaller than floating-point
2. **Power**: Less switching activity, lower power consumption
3. **Speed**: Fixed-point operations have predictable latency

Example: FIR filter coefficients often use Q15 format (1 sign bit, 15 fractional bits)

```cpp
sc_fixed<16, 1> coeff;    // Q15 coefficient
sc_fixed<32, 16> accum;   // Accumulator with guard bits
accum += coeff * sample;  // Multiply-accumulate
```

---

## 6. Datatype Conversion and Casting

### Automatic Conversions:

```cpp
sc_int<8> a = 5;           // int -> sc_int
sc_bv<8> b = a;           // sc_int -> sc_bv (bit pattern)
sc_uint<8> c = b;         // sc_bv -> sc_uint (unsigned interpretation)
```

### Explicit Conversion Methods:

| Method | Returns | Use Case |
|:-------|:--------|:---------|
| `to_int()` | int | C++ integer |
| `to_uint()` | unsigned int | C++ unsigned |
| `to_long()` | long | C++ long |
| `to_ulong()` | unsigned long | C++ unsigned long |
| `to_int64()` | int64 | 64-bit signed |
| `to_uint64()` | uint64 | 64-bit unsigned |
| `to_double()` | double | Floating-point |
| `to_string()` | std::string | Debug output |

### Bitwise Casting:

```cpp
sc_bv<32> bv = ...;
sc_int<32> si = bv;  // Interpret bits as signed integer
sc_uint<32> ui = bv; // Interpret bits as unsigned integer
```

This is equivalent to Verilog `$signed()` and `$unsigned()` system functions.

---

## 7. Summary: Datatype Selection Guide

| Requirement | Recommended Type | Why |
|:------------|:----------------|:----|
| Clock/Control | `bool` or `sc_signal<bool>` | Simple 2-value |
| Data bus with X/Z | `sc_lv<W>` or `sc_logic` | 4-value modeling |
| Optimized data bus | `sc_bv<W>` | 2-value, fast |
| 8-bit operations | `sc_int<8>` | Exact hardware match |
| Standard ALU | `sc_int<32>` | 32-bit processor |
| DSP/MAC | `sc_fixed<W,IWL>` | Fractional arithmetic |
| Cryptography | `sc_bigint<>` | Wide integers |
| Memory addressing | `sc_uint<32>` or `sc_uint<64>` | Address buses |

---

## 8. RTL Comparison Summary

| SystemC | Verilog | Notes |
|:--------|:--------|:------|
| `sc_bv<8>` | `wire [7:0]` | Bit vector |
| `sc_lv<8>` | `wire [7:0]` | With X/Z support |
| `sc_int<8>` | `reg signed [7:0]` | Signed integer |
| `sc_uint<8>` | `reg [7:0]` | Unsigned integer |
| `sc_logic` | `wire` | 4-value logic |
| `sc_fixed<16,8>` | N/A | C++ only, but synthesizable |
| `range(i,j)` | `[i:j]` | Bit slice |
| `and_reduce()` | `&bus` | Reduction AND |
