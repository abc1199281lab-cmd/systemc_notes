# SystemC Utils: Utils IDs (`sc_utils_ids`)

> **Source**: `ref/systemc/src/sysc/utils/sc_utils_ids.cpp`

## 1. Overview
This file serves as the **central registry** for report message text definitions in the utils, kernel, communication, bit, fx, int, and tracing modules. It links the integer/string IDs (declared in headers) to their actual human-readable warning/error messages.

## 2. Implementation Mechanism
-   **Macros**: Uses `SC_DEFINE_MESSAGE` macro to instantiate `extern const char id[] = text;` definitions.
-   **Inclusion**: It includes multiple `*_ids.h` files (like `sc_kernel_ids.h`, `sc_bit_ids.h`) while the macro is defined, effectively generating the definition code for all of them in this single translation unit.
-   **Registration**:
    -   Builds a static array `texts` of `sc_msg_def`.
    -   Uses a static `initialize()` function (triggered by the initialization of static variable `forty_two`) to register these messages with `sc_report_handler::add_static_msg_types`.

## 3. Environmental Overrides
The `initialize()` function also checks for `SC_DEPRECATION_WARNINGS`. If set to `DISABLE`, it suppresses IEEE 1666 deprecation warnings using `sc_report_handler::set_actions`.
