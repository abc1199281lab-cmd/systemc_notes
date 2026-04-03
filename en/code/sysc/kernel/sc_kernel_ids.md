# sc_kernel_ids.h - Core Error and Warning Message IDs

## Overview

`sc_kernel_ids.h` centrally defines all error and warning message IDs for the SystemC kernel. Each ID corresponds to a numeric code and a human-readable description string. These IDs are used by the `SC_REPORT_ERROR`, `SC_REPORT_WARNING`, and similar macros to produce diagnostic messages.

## Why is this file needed?

Just as hospitals have a unified disease code system (ICD), SystemC also needs a unified error code system. The benefits are:

1. **Programmatic handling**: Numeric IDs can be used to filter or specially handle certain errors
2. **Centralized management**: All error messages are in one place, making translation and modification convenient
3. **Avoiding duplicates**: Each error has a unique ID, preventing confusion

## Message Definition Mechanism

```cpp
#ifndef SC_DEFINE_MESSAGE
#define SC_DEFINE_MESSAGE(id, unused1, unused2) \
    namespace sc_core { extern SC_API const char id[]; }
#endif
```

The `SC_DEFINE_MESSAGE` macro has three parameters:
- `id`: C++ variable name (e.g., `SC_ID_COROUTINE_ERROR_`)
- `unused1`: Numeric code (e.g., 518)
- `unused2`: Error description string

In the header, this macro only declares an `extern const char[]`. The actual string definition is elsewhere (typically handled automatically by the report system).

## Message Category Overview

Message IDs range from 500-578, grouped by topic:

### Type and Operation Errors (500-504)

| ID | Code | Description |
|----|------|-------------|
| `SC_ID_NO_BOOL_RETURNED_` | 500 | Operator did not return boolean |
| `SC_ID_NO_INT_RETURNED_` | 501 | Operator did not return int |
| `SC_ID_NO_SC_LOGIC_RETURNED_` | 502 | Operator did not return sc_logic |
| `SC_ID_OPERAND_NOT_SC_LOGIC_` | 503 | Operand is not sc_logic |
| `SC_ID_OPERAND_NOT_BOOL_` | 504 | Operand is not bool |

### Object and Naming (505-506, 532-534)

| ID | Code | Description |
|----|------|-------------|
| `SC_ID_INSTANCE_EXISTS_` | 505 | Object already exists |
| `SC_ID_ILLEGAL_CHARACTERS_` | 506 | Illegal characters in name |
| `SC_ID_GEN_UNIQUE_NAME_` | 532 | Cannot generate unique name from empty string |
| `SC_ID_MODULE_NAME_STACK_EMPTY_` | 533 | Module name stack is empty |
| `SC_ID_NAME_EXISTS_` | 534 | Name already exists |

### Module Construction (509-513, 569)

| ID | Code | Description |
|----|------|-------------|
| `SC_ID_END_MODULE_NOT_CALLED_` | 509 | Module construction not properly completed (missing `sc_module_name` parameter) |
| `SC_ID_HIER_NAME_INCORRECT_` | 510 | Hierarchy name may be incorrect |
| `SC_ID_SET_STACK_SIZE_` | 511 | `set_stack_size()` can only be used with `SC_THREAD`/`SC_CTHREAD` |
| `SC_ID_SC_MODULE_NAME_USE_` | 512 | `sc_module_name` used incorrectly |
| `SC_ID_SC_MODULE_NAME_REQUIRED_` | 513 | Constructor requires `sc_module_name` parameter |
| `SC_ID_BAD_SC_MODULE_CONSTRUCTOR_` | 569 | Deprecated constructor style |

### Time Configuration (514-516, 567)

| ID | Code | Description |
|----|------|-------------|
| `SC_ID_SET_TIME_RESOLUTION_` | 514 | Failed to set time resolution |
| `SC_ID_SET_DEFAULT_TIME_UNIT_` | 515 | Failed to set default time unit |
| `SC_ID_DEFAULT_TIME_UNIT_CHANGED_` | 516 | Default time unit changed to time resolution |
| `SC_ID_TIME_CONVERSION_FAILED_` | 567 | sc_time conversion failed |

### Process Control (519-528, 537-543, 556-561, 563-564, 572-574)

| ID | Code | Description |
|----|------|-------------|
| `SC_ID_WAIT_NOT_ALLOWED_` | 519 | `wait()` can only be used in `SC_THREAD`/`SC_CTHREAD` |
| `SC_ID_NEXT_TRIGGER_NOT_ALLOWED_` | 520 | `next_trigger()` can only be used in `SC_METHOD` |
| `SC_ID_HALT_NOT_ALLOWED_` | 522 | `halt()` can only be used in `SC_CTHREAD` |
| `SC_ID_DONT_INITIALIZE_` | 524 | `dont_initialize()` has no effect on `SC_CTHREAD` |
| `SC_ID_WAIT_N_INVALID_` | 525 | `wait(n)` requires n > 0 |
| `SC_ID_WAIT_DURING_UNWINDING_` | 537 | Cannot call `wait()` during unwinding |
| `SC_ID_RETHROW_UNWINDING_` | 539 | `sc_unwind_exception` not re-thrown during kill/reset |
| `SC_ID_PROCESS_ALREADY_UNWINDING_` | 540 | Kill/reset ignored during unwinding |

### Simulation Control (544-549, 554)

| ID | Code | Description |
|----|------|-------------|
| `SC_ID_SIMULATION_TIME_OVERFLOW_` | 544 | Simulation time overflow |
| `SC_ID_SIMULATION_STOP_CALLED_TWICE_` | 545 | `sc_stop` called twice |
| `SC_ID_SIMULATION_START_AFTER_STOP_` | 546 | `sc_start` called after stop |
| `SC_ID_SIMULATION_START_AFTER_ERROR_` | 548 | Attempt to restart simulation after error |
| `SC_ID_SIMULATION_UNCAUGHT_EXCEPTION_` | 549 | Uncaught exception |

### Others

| ID | Code | Description |
|----|------|-------------|
| `SC_ID_INCONSISTENT_API_CONFIG_` | 517 | Inconsistent library configuration detected |
| `SC_ID_COROUTINE_ERROR_` | 518 | Coroutine package error |
| `SC_ID_IMMEDIATE_NOTIFICATION_` | 521 | Immediate notification not allowed during update phase or elaboration |
| `SC_ID_STAGE_CALLBACK_REGISTER_` | 552 | Stage callback registration related |
| `SC_ID_STAGE_CALLBACK_FORBIDDEN_` | 553 | Forbidden operation in stage callback |
| `SC_ID_INSERT_INITIALIZER_FN_` | 568 | Failed to insert initializer function |
| `SC_ID_UNMATCHED_SUSPENDABLE_` | 576 | Unmatched suspendable/unsuspendable request |

## Usage

```cpp
// In SystemC source code:
SC_REPORT_ERROR(SC_ID_WAIT_NOT_ALLOWED_, "");
SC_REPORT_WARNING(SC_ID_SIMULATION_STOP_CALLED_TWICE_, "");
```

## Related Files

- `sysc/utils/sc_report.h` - Report system (defines the `SC_DEFINE_MESSAGE` macro)
- `sc_simcontext.h` - Uses these IDs to generate simulation errors
- `sc_process_b.h` - Uses process-related error IDs
