# sc_int_ids.h — Integer Subsystem Error Reporting Identifiers

## Overview

`sc_int_ids.h` defines the error reporting identifiers used by the integer subsystem (`datatypes/int/`). When an error occurs during integer operations (such as initialization failure, assignment overflow, operation error, or type conversion failure), SystemC's reporting mechanism uses these identifiers to produce meaningful error messages.

**Source file:**
- `ref/systemc/src/sysc/datatypes/int/sc_int_ids.h`

## Everyday Analogy

This is like a "fault code table" in a factory. When a machine has a problem, it displays a code (e.g., E400), and the worker can look up the table to know what happened, rather than seeing a vague message like "machine broken."

## Defined Identifiers

| Identifier | Code | Description |
|------------|------|-------------|
| `SC_ID_INIT_FAILED_` | 400 | Initialization failed |
| `SC_ID_ASSIGNMENT_FAILED_` | 401 | Assignment failed |
| `SC_ID_OPERATION_FAILED_` | 402 | Operation failed |
| `SC_ID_CONVERSION_FAILED_` | 403 | Type conversion failed |

These identifiers have code numbers in the 400-499 range, reserved exclusively for the `datatypes/int` subsystem.

## Implementation Mechanism

Uses the `SC_DEFINE_MESSAGE` macro for declaration:

```cpp
SC_DEFINE_MESSAGE( SC_ID_INIT_FAILED_, 400, "initialization failed" )
```

This macro expands differently depending on the compilation environment, but the core functionality is to associate the identifier string with an error message so that the `sc_report` mechanism can use it.

## Related Files

- [sc_int_base.md](sc_int_base.md) — Uses these identifiers for error reporting
- [sc_uint_base.md](sc_uint_base.md) — Uses these identifiers for error reporting
- [sc_nbutils.md](sc_nbutils.md) — Uses these identifiers for error reporting
