# SystemC Data Types Overview - datatypes Directory

This directory contains all hardware simulation data type implementations in SystemC. These types are what distinguishes SystemC from ordinary C++ libraries: they allow software engineers to precisely describe digital signals in hardware circuits using C++.

## Why Do We Need Special Data Types?

Imagine you're building a house with LEGO bricks. Regular C++ types (`int`, `bool`) are like fixed-size LEGO bricks—you can only use 8-bit, 16-bit, 32-bit, or 64-bit. But in hardware design, you might need a 13-bit counter or a 128-bit bus. SystemC's data types are like LEGO bricks that can be cut to any size.

Furthermore, in real circuits, a wire doesn't just have two states of 0 and 1—it can also be in a "high impedance" (Z) or "unknown" (X) state. This is like a traffic light that, besides red and green, could also malfunction (unknown) or be completely off (high impedance).

## Subdirectory Overview

| Subdirectory | Purpose | Everyday Analogy |
|--------|------|----------|
| `bit/` | Bit and logic vector types | An array of switches—each switch can be on/off (two-valued) or on/off/disconnected/unknown (four-valued) |
| `int/` | Arbitrary-width integer types | A resizable numeric calculator, supporting signed/unsigned with any bit width |
| `fx/`  | Fixed-point number types | A hardware calculator with decimal points, used for DSP and scenarios requiring precise fractional arithmetic |
| `misc/`| Miscellaneous utility types | A utility toolbox containing value representation conversion and other common functions |

## Relationships Between Types

```mermaid
graph TB
    subgraph "bit/ - Bit and Logic Types"
        sc_bit["sc_bit<br/>(deprecated, use bool instead)"]
        sc_logic["sc_logic<br/>(four-valued logic: 0/1/X/Z)"]
        sc_bv["sc_bv&lt;W&gt;<br/>(bit vector)"]
        sc_lv["sc_lv&lt;W&gt;<br/>(logic vector)"]
        sc_proxy["sc_proxy&lt;T&gt;<br/>(shared vector interface)"]
    end

    subgraph "int/ - Integer Types"
        sc_int["sc_int&lt;W&gt; / sc_uint&lt;W&gt;<br/>(fixed width, ≤64 bits)"]
        sc_bigint["sc_bigint&lt;W&gt; / sc_biguint&lt;W&gt;<br/>(arbitrary width integers)"]
        sc_signed["sc_signed / sc_unsigned<br/>(dynamic width integers)"]
    end

    subgraph "fx/ - Fixed-Point Types"
        sc_fixed["sc_fixed&lt;W,I&gt;<br/>(constrained fixed-point)"]
        sc_fix["sc_fix<br/>(unconstrained fixed-point)"]
    end

    subgraph "misc/ - Miscellaneous"
        sc_value_base["sc_value_base<br/>(numeric type base class)"]
    end

    sc_bit --> sc_logic
    sc_logic --> sc_lv
    sc_bv --> sc_proxy
    sc_lv --> sc_proxy
    sc_bv -.->|"contains only 0/1"| sc_lv
    sc_lv -.->|"convertible"| sc_bigint
    sc_bv -.->|"convertible"| sc_bigint
    sc_fixed -.->|"uses internally"| sc_signed
    sc_value_base -.->|"base class"| sc_signed
```

## How to Choose the Right Type?

```mermaid
flowchart TD
    Start["What type do I need?"] --> Q1{"Need to represent<br/>X or Z states?"}
    Q1 -->|"Yes"| Q2{"Single bit<br/>or multi-bit?"}
    Q1 -->|"No"| Q3{"Need to do<br/>arithmetic?"}

    Q2 -->|"Single bit"| sc_logic_out["sc_logic"]
    Q2 -->|"Multi-bit"| sc_lv_out["sc_lv&lt;W&gt;"]

    Q3 -->|"No"| Q4{"Single bit<br/>or multi-bit?"}
    Q3 -->|"Yes"| Q5{"Need fractions?"}

    Q4 -->|"Single bit"| bool_out["bool"]
    Q4 -->|"Multi-bit"| sc_bv_out["sc_bv&lt;W&gt;"]

    Q5 -->|"No"| Q6{"More than 64 bits?"}
    Q5 -->|"Yes"| sc_fixed_out["sc_fixed / sc_fix"]

    Q6 -->|"No"| sc_int_out["sc_int / sc_uint"]
    Q6 -->|"Yes"| sc_bigint_out["sc_bigint / sc_biguint"]
```

## Related Files

- `bit/` - [Bit and logic types detailed documentation](bit/_index.md)
- `int/` - Integer types (sc_int, sc_uint, sc_bigint, sc_biguint, sc_signed, sc_unsigned)
- `fx/` - Fixed-point types (sc_fixed, sc_ufixed, sc_fix, sc_ufix)
- `misc/` - Miscellaneous utilities (sc_value_base, sc_concatref)
