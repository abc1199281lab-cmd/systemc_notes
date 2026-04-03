# sc_communication_ids -- Error/Warning Message IDs for the Communication Subsystem

## Overview

`sc_communication_ids.h` defines unique identifiers for all error and warning messages in the communication subsystem. These IDs are used with macros like `SC_REPORT_ERROR` and `SC_REPORT_WARNING`, allowing users to identify, filter, and handle specific types of errors.

**Source file:** `sc_communication_ids.h` (header-only)

## Everyday Analogy

Just like hospital disease classification codes (ICD codes), each error has a unique number. When you receive an error message, you can quickly look up the cause and solution by its number.

## Message ID List

All communication subsystem message IDs fall in the range **100-199**.

### Port-Related (100, 107-112)

| ID | Constant Name | Number | Description |
|----|--------------|--------|-------------|
| 100 | `SC_ID_PORT_OUTSIDE_MODULE_` | 100 | Port created outside a module |
| 107 | `SC_ID_BIND_IF_TO_PORT_` | 107 | Failed to bind interface to port |
| 108 | `SC_ID_BIND_PORT_TO_PORT_` | 108 | Failed to bind parent port to port |
| 109 | `SC_ID_COMPLETE_BINDING_` | 109 | Failed to complete binding |
| 110 | `SC_ID_INSERT_PORT_` | 110 | Failed to insert port |
| 111 | `SC_ID_REMOVE_PORT_` | 111 | Failed to remove port |
| 112 | `SC_ID_GET_IF_` | 112 | Failed to get interface |

### Clock-Related (101-103, 125, 128)

| ID | Constant Name | Number | Description |
|----|--------------|--------|-------------|
| 101 | `SC_ID_CLOCK_PERIOD_ZERO_` | 101 | sc_clock period is zero |
| 102 | `SC_ID_CLOCK_HIGH_TIME_ZERO_` | 102 | sc_clock high time is zero |
| 103 | `SC_ID_CLOCK_LOW_TIME_ZERO_` | 103 | sc_clock low time is zero |
| 125 | `SC_ID_ATTEMPT_TO_WRITE_TO_CLOCK_` | 125 | Attempt to write to sc_clock value |
| 128 | `SC_ID_ATTEMPT_TO_BIND_CLOCK_TO_OUTPUT_` | 128 | Attempt to bind sc_clock to output port |

### FIFO-Related (104-106)

| ID | Constant Name | Number | Description |
|----|--------------|--------|-------------|
| 104 | `SC_ID_MORE_THAN_ONE_FIFO_READER_` | 104 | sc_fifo cannot have multiple readers |
| 105 | `SC_ID_MORE_THAN_ONE_FIFO_WRITER_` | 105 | sc_fifo cannot have multiple writers |
| 106 | `SC_ID_INVALID_FIFO_SIZE_` | 106 | sc_fifo size must be at least 1 |

### Channel-Related (113-116)

| ID | Constant Name | Number | Description |
|----|--------------|--------|-------------|
| 113 | `SC_ID_INSERT_PRIM_CHANNEL_` | 113 | Failed to insert primitive channel |
| 114 | `SC_ID_REMOVE_PRIM_CHANNEL_` | 114 | Failed to remove primitive channel |
| 115 | `SC_ID_MORE_THAN_ONE_SIGNAL_DRIVER_` | 115 | sc_signal cannot have multiple drivers |
| 116 | `SC_ID_NO_DEFAULT_EVENT_` | 116 | Channel has no default event |

### Event Finder and Resolution-Related (117-118)

| ID | Constant Name | Number | Description |
|----|--------------|--------|-------------|
| 117 | `SC_ID_RESOLVED_PORT_NOT_BOUND_` | 117 | Resolved port not bound to resolved signal |
| 118 | `SC_ID_FIND_EVENT_` | 118 | Failed to find event |

### Semaphore (119)

| ID | Constant Name | Number | Description |
|----|--------------|--------|-------------|
| 119 | `SC_ID_INVALID_SEMAPHORE_VALUE_` | 119 | sc_semaphore initial value must be >= 0 |

### Export-Related (120-124, 126)

| ID | Constant Name | Number | Description |
|----|--------------|--------|-------------|
| 120 | `SC_ID_SC_EXPORT_HAS_NO_INTERFACE_` | 120 | sc_export has no interface |
| 121 | `SC_ID_INSERT_EXPORT_` | 121 | Failed to insert sc_export |
| 122 | `SC_ID_EXPORT_OUTSIDE_MODULE_` | 122 | sc_export created outside a module |
| 123 | `SC_ID_SC_EXPORT_NOT_REGISTERED_` | 123 | Removing unregistered sc_export |
| 124 | `SC_ID_SC_EXPORT_NOT_BOUND_AFTER_CONSTRUCTION_` | 124 | sc_export not bound after construction |
| 126 | `SC_ID_SC_EXPORT_ALREADY_BOUND_` | 126 | sc_export already bound |

### Other (127, 129)

| ID | Constant Name | Number | Description |
|----|--------------|--------|-------------|
| 127 | `SC_ID_INVALID_HIERARCHICAL_BIND_` | 127 | Multi-target socket binding error |
| 129 | `SC_ID_INSERT_STUB_` | 129 | Failed to insert sc_stub |

## Definition Mechanism

```cpp
#define SC_DEFINE_MESSAGE(id, unused1, unused2) \
    namespace sc_core { extern SC_API const char id[]; }
```

Each message ID is defined as an `extern const char[]` array. The actual string content is defined in the corresponding `.cpp` file. The second parameter of the macro is the number (for uniqueness), and the third parameter is a human-readable description.

## Usage

Used in code via `SC_REPORT_ERROR` or `SC_REPORT_WARNING`:

```cpp
// example from sc_interface.cpp
SC_REPORT_WARNING( SC_ID_NO_DEFAULT_EVENT_, 0 );

// example from sc_port.cpp
SC_REPORT_ERROR( SC_ID_BIND_IF_TO_PORT_, "simulation running" );
```

Users can customize error handling behavior through `sc_report_handler`, for example converting certain errors to warnings, or completely suppressing certain messages.

## Design Notes

### Why use string constants instead of enums?

String constants can carry more semantic information and are directly readable in error reports. The `SC_REPORT_ERROR` system outputs both the ID string and the additional message, making debugging easier.

### Number Ranges

Different subsystems use different number ranges:
- 100-199: Communication subsystem (this file)
- Other ranges are used by the kernel, datatypes, and other subsystems

## Related Files

- `sc_report.h` - Error reporting infrastructure
- `sc_port.cpp` - Uses these IDs to report binding errors
- `sc_export.cpp` - Uses these IDs to report export errors
- `sc_signal.cpp` - Uses these IDs to report signal errors
- `sc_clock.cpp` - Uses these IDs to report clock errors
