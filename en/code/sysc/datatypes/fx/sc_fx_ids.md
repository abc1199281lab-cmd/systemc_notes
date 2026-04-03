# sc_fx_ids.h -- Fixed-Point Error Message IDs

## Overview

`sc_fx_ids.h` defines all **error report IDs** for the fixed-point subsystem. These IDs work with the SystemC error reporting mechanism (`sc_report`) so that users see clear error messages when debugging.

## Everyday Analogy

Like the error codes on a car's dashboard -- engine fault is P0300, oil pressure anomaly is P0520. Each error code corresponds to a specific problem description, allowing the mechanic to quickly locate the issue.

## Error ID List

All IDs are numbered in the **300-399** range (allocated to the datatypes/fx module).

| ID | Number | Error Message | Trigger Condition |
|----|--------|---------------|-------------------|
| `SC_ID_INVALID_WL_` | 300 | "total wordlength <= 0 is not valid" | Total word length set to 0 or negative |
| `SC_ID_INVALID_N_BITS_` | 301 | "number of bits < 0 is not valid" | Saturation bits is negative |
| `SC_ID_INVALID_DIV_WL_` | 302 | "division wordlength <= 0 is not valid" | Division word length set to 0 or negative |
| `SC_ID_INVALID_CTE_WL_` | 303 | "constant wordlength <= 0 is not valid" | Constant word length set to 0 or negative |
| `SC_ID_INVALID_MAX_WL_` | 304 | "maximum wordlength <= 0 and != -1 is not valid" | Maximum word length is invalid |
| `SC_ID_INVALID_FX_VALUE_` | 305 | "invalid fixed-point value" | Value is NaN or Inf |
| `SC_ID_INVALID_O_MODE_` | 306 | "invalid overflow mode" | Overflow mode is invalid |
| `SC_ID_OUT_OF_RANGE_` | 307 | "index out of range" | Bit index is out of range |
| `SC_ID_CONTEXT_BEGIN_FAILED_` | 308 | "context begin failed" | Duplicate call to context begin |
| `SC_ID_CONTEXT_END_FAILED_` | 309 | "context end failed" | Calling end on an unstarted context |
| `SC_ID_WRAP_SM_NOT_DEFINED_` | 310 | "SC_WRAP_SM not defined for unsigned numbers" | Using sign magnitude wrap on unsigned numbers |

## Implementation Mechanism

Each ID is defined using the `SC_DEFINE_MESSAGE` macro:

```cpp
SC_DEFINE_MESSAGE( SC_ID_INVALID_WL_, 300,
    "total wordlength <= 0 is not valid" )
```

This macro declares an `extern const char[]` in the `sc_core` namespace; the actual string definition resides in `sc_report_handler.cpp`.

## Usage

These IDs are used through the validation macros in `sc_fxdefs.h`:

```cpp
// In sc_fxdefs.h:
#define SC_CHECK_WL_(wl) \
    SC_ERROR_IF_( (wl) <= 0, sc_core::SC_ID_INVALID_WL_ )

// Usage in code:
sc_fxtype_params::wl( int wl_ ) {
    SC_CHECK_WL_( wl_ );  // triggers error if wl_ <= 0
    m_wl = wl_;
}
```

## Related Files

- `sc_fxdefs.h` -- Validation macros that use these IDs
- `sysc/utils/sc_report.h` -- Error reporting infrastructure
- `sc_context.h` -- Uses `SC_ID_CONTEXT_BEGIN_FAILED_` / `SC_ID_CONTEXT_END_FAILED_`
- `scfx_params.h` -- Uses `SC_ID_INVALID_O_MODE_`
