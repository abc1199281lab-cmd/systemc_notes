# Data Types

## Real-Life Analogy: Different Rulers and Measuring Tools

Imagine you are in a hardware store, with various measuring tools in front of you:

- **C++ `int`** = A regular ruler -- can measure most things, but with limited precision and range
- **`sc_int<N>`** = A custom-length ruler -- you can specify exactly how many centimeter marks you need
- **`sc_bigint<N>`** = An architect's tape measure -- can measure very long distances
- **`sc_logic`** = A traffic light -- not just red/green, but also "yellow" and "flashing malfunction"
- **`sc_bv<N>`** = A row of traffic lights -- multiple signals lined up together
- **`sc_fixed<>`** = A precision scale -- can measure to many decimal places

Hardware needs these special types because "numbers" in hardware are very different from "numbers" in software.

---

## Why Does Hardware Need Special Types?

### Software vs Hardware View of Numbers

```mermaid
graph LR
    subgraph "Software World"
        S1["int → Fixed 32 or 64 bits"]
        S2["Only 0 and 1"]
        S3["float/double for decimals"]
    end

    subgraph "Hardware World"
        H1["Arbitrary bit widths<br/>3-bit, 7-bit, 13-bit..."]
        H2["Four values: 0, 1, X, Z"]
        H3["Fixed-point<br/>Precise control of precision"]
    end

    S1 -.->|"Not flexible enough"| H1
    S2 -.->|"Not complete enough"| H2
    S3 -.->|"Not precise enough"| H3

    style S1 fill:#ffcdd2
    style S2 fill:#ffcdd2
    style S3 fill:#ffcdd2
    style H1 fill:#c8e6c9
    style H2 fill:#c8e6c9
    style H3 fill:#c8e6c9
```

| Requirement | C++ Native Type | SystemC Type | Reason |
|-------------|----------------|--------------|--------|
| 5-bit counter | None | `sc_uint<5>` | The hardware register is exactly 5 bits |
| Unknown value X | None | `sc_logic` | Value is unknown before initialization |
| High-impedance Z | None | `sc_logic` | Needed for tri-state buses |
| 128-bit width | None | `sc_biguint<128>` | Wide data paths |
| Precise decimals | `double` has rounding errors | `sc_fixed<8,4>` | DSP requires precise fixed-point numbers |

---

## Two-Value Logic vs Four-Value Logic

### Two-Value Logic (`sc_bit`, `sc_bv`)

Only `0` and `1`, just like ordinary boolean values.

### Four-Value Logic (`sc_logic`, `sc_lv`)

Four possible values, corresponding to real electrical states in hardware:

```mermaid
graph TD
    subgraph "Four-Value Logic"
        V0["'0' — Logic Low<br/>Definitely 0"]
        V1["'1' — Logic High<br/>Definitely 1"]
        VX["'X' — Unknown<br/>Could be 0 or 1"]
        VZ["'Z' — High Impedance<br/>No driver (open circuit)"]
    end

    style V0 fill:#c8e6c9
    style V1 fill:#c8e6c9
    style VX fill:#ffcdd2
    style VZ fill:#fff3e0
```

### When Do X and Z Appear?

```mermaid
flowchart TD
    X_CASE["When does X appear?"]
    X1["Register not yet initialized"]
    X2["Two drivers conflict<br/>One outputs 0, the other outputs 1"]
    X3["Timing violation<br/>Setup time or hold time not met"]

    Z_CASE["When does Z appear?"]
    Z1["Tri-state buffer disabled<br/>(output enable = 0)"]
    Z2["No device driving<br/>the bus"]
    Z3["IC pin floating<br/>(not connected)"]

    X_CASE --> X1
    X_CASE --> X2
    X_CASE --> X3

    Z_CASE --> Z1
    Z_CASE --> Z2
    Z_CASE --> Z3

    style X_CASE fill:#ffcdd2
    style Z_CASE fill:#fff3e0
```

### Four-Value Logic Truth Table

AND operation:

| & | 0 | 1 | X | Z |
|---|---|---|---|---|
| **0** | 0 | 0 | 0 | 0 |
| **1** | 0 | 1 | X | X |
| **X** | 0 | X | X | X |
| **Z** | 0 | X | X | X |

**Intuitive understanding**: As long as one side is definitely 0, the AND result is definitely 0.
If anything is uncertain (X or Z), the result is also uncertain.

---

## Fixed-Width Integers

### sc_int / sc_uint (1 to 64 bits)

```mermaid
graph TD
    subgraph "sc_int / sc_uint Family"
        SI["sc_int<N><br/>Signed integer<br/>1 ≤ N ≤ 64"]
        SU["sc_uint<N><br/>Unsigned integer<br/>1 ≤ N ≤ 64"]
    end

    subgraph "Underlying Implementation"
        NATIVE["Uses C++ native int64<br/>for storage and arithmetic<br/>→ Fast!"]
    end

    SI --> NATIVE
    SU --> NATIVE

    style SI fill:#e3f2fd
    style SU fill:#e3f2fd
    style NATIVE fill:#c8e6c9
```

```cpp
sc_uint<4> nibble = 0xF;    // 4-bit unsigned: 0~15
sc_int<8>  byte_val = -128; // 8-bit signed: -128~127
sc_uint<1> single_bit = 1;  // 1-bit: just a single bit

// Bit operations
nibble[2] = 0;               // Set bit 2 to 0
sc_uint<2> sub = nibble.range(3, 2); // Extract upper 2 bits
```

### sc_bigint / sc_biguint (Arbitrary Width)

```mermaid
graph TD
    subgraph "sc_bigint / sc_biguint Family"
        BI["sc_bigint<N><br/>Signed integer<br/>N can be any positive integer"]
        BU["sc_biguint<N><br/>Unsigned integer<br/>N can be any positive integer"]
    end

    subgraph "Underlying Implementation"
        ARRAY["Uses integer array to emulate<br/>large number arithmetic<br/>→ Slower than sc_int"]
    end

    BI --> ARRAY
    BU --> ARRAY

    style BI fill:#fff3e0
    style BU fill:#fff3e0
    style ARRAY fill:#ffcdd2
```

```cpp
sc_biguint<128> wide_data;    // 128-bit unsigned
sc_bigint<256>  very_wide;    // 256-bit signed
sc_biguint<1024> huge;        // 1024-bit is fine too!
```

### Selection Guide

```mermaid
flowchart TD
    START["I need an integer"] --> Q1{"How many bits?"}

    Q1 -->|"≤ 64 bit"| Q2{"Need X/Z values?"}
    Q2 -->|"No"| SMALL["sc_int<N> / sc_uint<N><br/>Fastest!"]
    Q2 -->|"Yes"| LV["sc_lv<N><br/>Four-value logic vector"]

    Q1 -->|"> 64 bit"| Q3{"Need X/Z values?"}
    Q3 -->|"No"| BIG["sc_bigint<N> / sc_biguint<N>"]
    Q3 -->|"Yes"| LV2["sc_lv<N><br/>Four-value logic vector"]

    style SMALL fill:#c8e6c9
    style BIG fill:#fff3e0
    style LV fill:#e3f2fd
    style LV2 fill:#e3f2fd
```

---

## Bit Vectors and Logic Vectors

### sc_bv -- Two-Value Bit Vector

```cpp
sc_bv<8> byte_vec = "10110011";   // 8-bit two-value vector
sc_bv<4> nibble = byte_vec.range(7, 4);  // Extract upper 4 bits
byte_vec[0] = 1;                  // Set the lowest bit
```

### sc_lv -- Four-Value Logic Vector

```cpp
sc_lv<8> logic_vec = "10XZ10X1";  // Contains X and Z!
sc_lv<4> bus_val = "ZZZZ";        // High-impedance bus
```

```mermaid
graph LR
    subgraph "sc_bv<8>"
        B0["1"]
        B1["0"]
        B2["1"]
        B3["1"]
        B4["0"]
        B5["0"]
        B6["1"]
        B7["1"]
    end

    subgraph "sc_lv<8>"
        L0["1"]
        L1["0"]
        L2["X"]
        L3["Z"]
        L4["1"]
        L5["0"]
        L6["X"]
        L7["1"]
    end

    style B0 fill:#c8e6c9
    style B6 fill:#c8e6c9
    style L2 fill:#ffcdd2
    style L3 fill:#fff3e0
    style L6 fill:#ffcdd2
```

---

## Fixed-Point Numbers

### What Are Fixed-Point Numbers?

In floating-point numbers (`float`/`double`), the decimal point position "floats."
In fixed-point numbers, the decimal point position is "fixed."

```mermaid
graph TD
    subgraph "Floating-Point float"
        F1["3.14 → Decimal point after digit 1"]
        F2["0.00314 → Decimal point after digit 0"]
        F3["3140.0 → Decimal point after digit 4"]
        NOTE_F["Decimal point position changes!"]
    end

    subgraph "Fixed-Point sc_fixed<8,4>"
        FX1["0011.1001 → Decimal point always after bit 4"]
        FX2["Total 8 bits, 4 integer bits, 4 fractional bits"]
        NOTE_FX["Decimal point position is fixed!"]
    end

    style NOTE_F fill:#fff3e0
    style NOTE_FX fill:#c8e6c9
```

### sc_fixed Parameters

```cpp
sc_fixed<WL, IWL, Q_MODE, O_MODE, N_BITS> value;
//        |    |     |       |       |
//        |    |     |       |       Number of saturated bits
//        |    |     |       Overflow mode (SC_SAT, SC_WRAP...)
//        |    |     Quantization mode (SC_RND, SC_TRN...)
//        |    Integer word length
//        Total word length
```

```mermaid
graph TD
    subgraph "Bit Layout of sc_fixed<8, 4>"
        direction LR
        I3["bit 7<br/>Integer"]
        I2["bit 6<br/>Integer"]
        I1["bit 5<br/>Integer"]
        I0["bit 4<br/>Integer"]
        DOT["Decimal<br/>Point"]
        F0["bit 3<br/>Fraction"]
        F1["bit 2<br/>Fraction"]
        F2["bit 1<br/>Fraction"]
        F3["bit 0<br/>Fraction"]
    end

    I3 --- I2 --- I1 --- I0 --- DOT --- F0 --- F1 --- F2 --- F3

    style I3 fill:#e3f2fd
    style I2 fill:#e3f2fd
    style I1 fill:#e3f2fd
    style I0 fill:#e3f2fd
    style DOT fill:#ffcdd2
    style F0 fill:#c8e6c9
    style F1 fill:#c8e6c9
    style F2 fill:#c8e6c9
    style F3 fill:#c8e6c9
```

### Quantization and Overflow Modes

```mermaid
graph TD
    subgraph "Quantization Modes (when fractional precision is insufficient)"
        Q1["SC_RND — Round"]
        Q2["SC_TRN — Truncate (round toward negative infinity)"]
        Q3["SC_RND_ZERO — Round toward zero"]
        Q4["SC_RND_INF — Round toward infinity"]
    end

    subgraph "Overflow Modes (when integer exceeds range)"
        O1["SC_SAT — Saturate (clamp to max/min value)"]
        O2["SC_WRAP — Wrap around (like counter overflow)"]
        O3["SC_SAT_ZERO — Saturate to zero"]
    end

    style Q1 fill:#e3f2fd
    style Q2 fill:#e3f2fd
    style O1 fill:#fff3e0
    style O2 fill:#fff3e0
```

### Why Use Fixed-Point Instead of Floating-Point?

| Property | Floating-Point | Fixed-Point |
|----------|---------------|-------------|
| Hardware cost | High (requires complex FPU) | Low (only needs integer ALU) |
| Precision control | Precision varies with magnitude | Fixed and predictable precision |
| Speed | Slower | Faster |
| Use cases | General-purpose computing | DSP, audio, image processing |

---

## Type System Overview

```mermaid
classDiagram
    class sc_value_base {
        <<abstract>>
    }

    class sc_int_base {
        -m_val : int64
        -m_len : int
    }

    class sc_uint_base {
        -m_val : uint64
        -m_len : int
    }

    class sc_signed {
        -digit : sc_digit*
        -ndigits : int
        -nbits : int
    }

    class sc_unsigned {
        -digit : sc_digit*
        -ndigits : int
        -nbits : int
    }

    class sc_lv_base {
        -m_data : sc_digit*
        -m_ctrl : sc_digit*
        -m_len : int
    }

    class sc_bv_base {
        -m_data : sc_digit*
        -m_len : int
    }

    class sc_fxnum {
        -m_rep : scfx_rep*
        -m_params : sc_fxtype_params
    }

    sc_value_base <|-- sc_int_base
    sc_value_base <|-- sc_uint_base
    sc_value_base <|-- sc_signed
    sc_value_base <|-- sc_unsigned

    sc_int_base <|-- "sc_int<N>"
    sc_uint_base <|-- "sc_uint<N>"
    sc_signed <|-- "sc_bigint<N>"
    sc_unsigned <|-- "sc_biguint<N>"

    sc_bv_base <|-- "sc_bv<N>"
    sc_lv_base <|-- "sc_lv<N>"
    sc_bv_base <|-- sc_lv_base : extends

    sc_fxnum <|-- "sc_fixed<...>"
    sc_fxnum <|-- "sc_ufixed<...>"
```

---

## Related Modules

| Concept | File | Relationship |
|---------|------|-------------|
| Communication | [communication.md](communication.md) | Signal template parameters use these types |
| Waveform Tracing | [tracing.md](tracing.md) | Tracing records value changes of these types |
| Module Hierarchy | [hierarchy.md](hierarchy.md) | Port template parameters also use these types |

### Corresponding Source Code Files

| Source Code Concept | Code File |
|--------------------|-----------|
| sc_int / sc_uint | [doc_v2/code/sysc/datatypes/int/_index.md](../code/sysc/datatypes/int/_index.md) |
| sc_bigint / sc_biguint | [doc_v2/code/sysc/datatypes/int/_index.md](../code/sysc/datatypes/int/_index.md) |
| sc_bit | [doc_v2/code/sysc/datatypes/bit/sc_bit.md](../code/sysc/datatypes/bit/sc_bit.md) |
| sc_logic | [doc_v2/code/sysc/datatypes/bit/sc_logic.md](../code/sysc/datatypes/bit/sc_logic.md) |
| sc_bv | [doc_v2/code/sysc/datatypes/bit/sc_bv.md](../code/sysc/datatypes/bit/sc_bv.md) |
| sc_lv | [doc_v2/code/sysc/datatypes/bit/sc_lv.md](../code/sysc/datatypes/bit/sc_lv.md) |
| sc_fixed | [doc_v2/code/sysc/datatypes/fx/sc_fixed.md](../code/sysc/datatypes/fx/sc_fixed.md) |
| sc_fxnum | [doc_v2/code/sysc/datatypes/fx/sc_fxnum.md](../code/sysc/datatypes/fx/sc_fxnum.md) |

---

## Learning Tips

1. **Not sure which to pick? Start with `sc_uint<N>`** -- the most commonly used and fastest, suitable for most situations
2. **Only use `sc_logic` / `sc_lv` when you need X/Z values** -- four-value logic operations are slower
3. **`sc_int<64>` is much faster than `sc_bigint<64>`** -- use `sc_int`/`sc_uint` for anything 64 bits or fewer
4. **Fixed-point is an advanced topic** -- beginners can skip it and come back when DSP modeling is needed
5. **Bit operations and bit slicing are very handy** -- `x.range(7,4)` and `x[3]` are commonly used operations
6. **Watch out for signed vs unsigned** -- mixing them produces unexpected results, just like in C++
