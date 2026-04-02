# sc_fxtype_params.h / .cpp -- Fixed-Point Type Parameters

## Overview

`sc_fxtype_params` encapsulates the five core parameters of a fixed-point type: word length (wl), integer word length (iwl), quantization mode (q_mode), overflow mode (o_mode), and saturation bits (n_bits). It is the central data structure for "format description" of fixed-point numbers.

## Everyday Analogy

If a fixed-point number is a photograph, `sc_fxtype_params` is the camera settings:
- **Word length (wl)** = resolution (total number of pixels)
- **Integer word length (iwl)** = focal length (determines how far the scene can reach)
- **Quantization mode (q_mode)** = compression algorithm (JPEG quality setting)
- **Overflow mode (o_mode)** = overexposure handling (saturate to white vs. auto-adjust)
- **Saturation bits (n_bits)** = special exposure parameter

## Class Details

### Member Variables

| Member | Type | Description |
|--------|------|-------------|
| `m_wl` | `int` | Total word length, must be > 0 |
| `m_iwl` | `int` | Integer word length, can be any integer |
| `m_q_mode` | `sc_q_mode` | Quantization mode |
| `m_o_mode` | `sc_o_mode` | Overflow mode |
| `m_n_bits` | `int` | Saturation bits, must be >= 0 |

**Note:** `iwl` can be greater than `wl` (meaning no fractional bits) or negative (meaning all bits are after the decimal point). Fractional word length = `wl - iwl`.

### Constructors

| Constructor | Behavior |
|-------------|----------|
| `sc_fxtype_params()` | Gets defaults from `sc_fxtype_context` |
| `sc_fxtype_params(wl, iwl)` | Specifies word lengths, rest from defaults |
| `sc_fxtype_params(q, o, n)` | Specifies modes, word lengths from defaults |
| `sc_fxtype_params(wl, iwl, q, o, n)` | Fully specifies all parameters |
| `sc_fxtype_params(params, wl, iwl)` | Copy and override word lengths |
| `sc_fxtype_params(params, q, o, n)` | Copy and override modes |
| `sc_fxtype_params(sc_without_context)` | Uses compile-time defaults |

### Word Length Meaning

```
    Integer part (iwl bits)       Fractional part (wl - iwl bits)
    <---------------------->     <------------------------------>
    [sign] [integer bits] . [fractional bits]
    <---------------------- wl bits -------------------------->
```

Example: `sc_fixed<8, 4>` means:
- wl = 8 (8 bits total)
- iwl = 4 (4 integer bits, including sign bit)
- Fractional part = 4 bits
- Representable range: -8.0 to +7.9375 (step 0.0625 = 2^-4)

### `sc_fxtype_context`

```cpp
typedef sc_context<sc_fxtype_params> sc_fxtype_context;
```

Usage example:

```cpp
{
    sc_fxtype_context ctx(sc_fxtype_params(16, 8, SC_RND, SC_SAT));
    // All sc_fix/sc_ufix created here default to 16-bit, 8-integer-bit
    sc_fix a;  // wl=16, iwl=8, SC_RND, SC_SAT
    sc_fix b;  // same defaults
}
```

## .cpp File

`sc_fxtype_params.cpp` contains:

1. Template instantiation of `sc_global<sc_fxtype_params>` and `sc_context<sc_fxtype_params>`
2. `to_string()` -- output format like `"(16,8,SC_RND,SC_SAT,0)"`
3. `print()` -- same as `to_string()`
4. `dump()` -- verbose output of each field

## Related Files

- `sc_fxdefs.h` -- Enumeration and default value definitions
- `sc_context.h` -- `sc_context<T>` template
- `scfx_params.h` -- Used in combination within `scfx_params`
- `sc_fxnum.h` -- Uses `sc_fxtype_params` to parameterize `sc_fxnum`
