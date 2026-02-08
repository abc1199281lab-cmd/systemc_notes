# SystemC Data Types: Fixed Precision Integers (`sc_int_base`, `sc_uint_base`)

> **Source**: `ref/systemc/src/sysc/datatypes/int/sc_int_base.cpp`, `ref/systemc/src/sysc/datatypes/int/sc_uint_base.cpp`

## 1. Overview
These classes form the basis of `sc_int<W>` and `sc_uint<W>`. They are optimized for widths up to 64 bits (`SC_INTWIDTH`).

## 2. Implementation
-   **Storage**: They rely on native C++ types `int64` and `uint64` (`m_val`) for storage and arithmetic. This makes them significantly faster than arbitrary-precision types (`sc_bigint`).
-   **Width Management**:
    -   `m_len`: Bit width of the type.
    -   `m_ulen`: Unused bits (`64 - m_len`), used for sign extension or masking.

## 3. Key Mechanisms

### 3.1 Arithmetic
Arithmetic operations (`+`, `-`, `*`, etc.) translate directly to native C++ operations, which is efficient.

### 3.2 Part Selection (`subref`)
-   Accessing a slice of bits returns a proxy object (`sc_int_subref` / `sc_uint_subref`).
-   To handle part selections effectively (e.g., across 64-bit boundaries or for concatenation), they implement `concat_get_ctrl` and `concat_get_data` methods, though `sc_int` doesn't strictly need control bits (it's 2-valued).

## 4. Usage
-   Use `sc_int<W>` and `sc_uint<W>` whenever `W <= 64` for best simulation performance.
-   Be aware that they wrap around 64-bit arithmetic semantics.
