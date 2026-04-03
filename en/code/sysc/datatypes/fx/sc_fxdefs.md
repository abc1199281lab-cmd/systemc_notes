# sc_fxdefs.h / sc_fxdefs.cpp -- Fixed-Point Definitions and Enumerations

## Overview

`sc_fxdefs.h` defines all the **enumerations**, **default constants**, and **validation macros** for the fixed-point system. It is the "specification document" of the entire fixed-point subsystem, defining the names and default values for all behavior modes.

## Everyday Analogy

This file is like a "menu" or "settings panel." Just as your phone's display settings have options like "brightness," "color temperature," and "font size," `sc_fxdefs.h` lists all the configurable "settings" for fixed-point numbers.

## Enumerations

### `sc_enc` -- Encoding

Determines the sign representation of values:

| Value | Meaning | Description |
|-------|---------|-------------|
| `SC_TC_` | Two's Complement | Signed, like everyday positive and negative numbers |
| `SC_US_` | Unsigned | Only positive numbers and zero |

### `sc_q_mode` -- Quantization Mode

How to handle excess bits when the fractional part exceeds the representable range:

| Value | Meaning | Analogy |
|-------|---------|---------|
| `SC_RND` | Round to plus infinity | Standard rounding (biased toward positive infinity) |
| `SC_RND_ZERO` | Round to zero | Round toward zero |
| `SC_RND_MIN_INF` | Round to minus infinity | Round toward negative infinity |
| `SC_RND_INF` | Round to infinity | Round away from zero |
| `SC_RND_CONV` | Convergent rounding | Banker's rounding (round to nearest even) |
| `SC_TRN` | Truncation | Truncate directly (toward negative infinity) |
| `SC_TRN_ZERO` | Truncation to zero | Truncate directly (toward zero) |

**Banker's rounding analogy:** Imagine you toss loose change into a piggy bank every day. If you always round up, over time you'll save slightly more (because 0.5 always rounds up). Banker's rounding chooses to round to the nearest even number when at 0.5, which is fairer over the long term.

### `sc_o_mode` -- Overflow Mode

What to do when a value exceeds the representable range:

| Value | Meaning | Analogy |
|-------|---------|---------|
| `SC_SAT` | Saturation | Clamp at the max/min value (like a volume knob turned all the way) |
| `SC_SAT_ZERO` | Saturation to zero | Set directly to zero |
| `SC_SAT_SYM` | Symmetrical saturation | Symmetric saturation (positive and negative ranges are equal) |
| `SC_WRAP` | Wrap-around | Wrap around (like an odometer going from 999999 to 000000) |
| `SC_WRAP_SM` | Sign magnitude wrap-around | Sign-magnitude wrap-around |

### `sc_switch` -- Switch

```cpp
enum sc_switch { SC_OFF, SC_ON };
```

Used to control enabling/disabling of the cast switch.

### `sc_fmt` -- Output Format

```cpp
enum sc_fmt { SC_F, SC_E };
```

| Value | Meaning | Example |
|-------|---------|---------|
| `SC_F` | Fixed format | `3.14159` |
| `SC_E` | Scientific format | `3.14159e+0` |

## Default Constants

### Type Parameter Defaults

| Constant | Value | Description |
|----------|-------|-------------|
| `SC_BUILTIN_WL_` | 32 | Default total word length |
| `SC_BUILTIN_IWL_` | 32 | Default integer word length |
| `SC_BUILTIN_Q_MODE_` | `SC_TRN` | Default quantization mode (truncation) |
| `SC_BUILTIN_O_MODE_` | `SC_WRAP` | Default overflow mode (wrap-around) |
| `SC_BUILTIN_N_BITS_` | 0 | Default saturation bits |
| `SC_BUILTIN_CAST_SWITCH_` | `SC_ON` | Default cast enabled |

### Value Type Defaults

| Constant | Value | Description |
|----------|-------|-------------|
| `SC_BUILTIN_DIV_WL_` | 64 | Division operation word length |
| `SC_BUILTIN_CTE_WL_` | 64 | Constant word length |
| `SC_BUILTIN_MAX_WL_` | 1024 | Maximum allowed word length |

These values can be overridden at compile time via the `SC_FXDIV_WL`, `SC_FXCTE_WL`, `SC_FXMAX_WL` macros.

## Validation Macros

```cpp
SC_CHECK_WL_(wl)       // wl > 0
SC_CHECK_N_BITS_(n)    // n >= 0
SC_CHECK_DIV_WL_(wl)   // wl > 0
SC_CHECK_CTE_WL_(wl)   // wl > 0
SC_CHECK_MAX_WL_(wl)   // wl > 0 or wl == -1
```

These macros trigger `SC_REPORT_ERROR` and abort the program when a parameter is invalid.

## Observer Macros

```cpp
SC_OBSERVER_(object, observer_type, event)
SC_OBSERVER_DEFAULT_(observer_type)
```

Provides generic macros for the observer pattern, used to notify observers when fixed-point objects are read or written.

## sc_fxdefs.cpp

The `.cpp` file implements `to_string()` functions for each enumeration, converting enum values to readable strings (e.g., `SC_TC_` -> `"SC_TC_"`).

## Related Files

- `sc_fx_ids.h` -- Error message IDs (used by validation macros)
- `sc_fxtype_params.h` -- Uses these enumerations and constants
- `sc_fxcast_switch.h` -- Uses the `sc_switch` enumeration
- `sc_fxnum_observer.h` -- Uses observer macros
