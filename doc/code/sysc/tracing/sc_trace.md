# SystemC Tracing: Trace API (`sc_trace`)

> **Source**: `ref/systemc/src/sysc/tracing/sc_trace.cpp`

## 1. Overview
This file implements the `sc_trace` function overrides. These are the user-facing APIs called to add signals or variables to a trace file.

## 2. Functionality
The `sc_trace` functions are essentially wrappers. They take an object (like `sc_signal`, `int`, `bool`, etc.) and a pointer to an `sc_trace_file`, then delegate the actual tracing registration to the trace file object.

```cpp
sc_trace( sc_trace_file* tf, const sc_signal_in_if<char>& object, ... ) {
    if( tf ) {
        tf->trace( object.read(), name, width );
    }
}
```

## 3. Supported Types
It defines macros (`DEFN_TRACE_FUNC...`) to generate `sc_trace` overrides for:
-   **Native C++ types**: `bool`, `float`, `double`, `int`, `long`, `short`, `char`, etc.
-   **SystemC types**: `sc_dt::sc_logic`, `sc_dt::sc_bit`, `sc_dt::sc_int`, `sc_dt::sc_uint`, etc.
-   **Fixed Point**: `sc_fxval`, `sc_fxnum` (if `SC_INCLUDE_FX` is defined).

## 4. Helper Functions
-   `tprintf`: A utility logic to format and write comments to the trace file.
