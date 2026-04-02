# sc_tracing_ids.h - Tracing Subsystem Error Message IDs

> Defines all unique identifiers for error, warning, and informational messages used by the tracing subsystem. ID range is 700-799.

## Everyday Analogy

This file is like a hospital's "diagnostic code table". Each code corresponds to a specific "symptom" -- when the tracing system has a problem, it doesn't just say "broken"; instead, it gives a precise code (e.g., 701 = "cannot open file"), helping engineers quickly locate the issue.

## Overview

Messages are defined through the `SC_DEFINE_MESSAGE` macro. Each message consists of three parts:
1. **Symbolic constant** (e.g., `SC_ID_TRACING_FOPEN_FAILED_`) -- used in code
2. **Numeric ID** (e.g., 701) -- used for filtering and identification
3. **Default message text** (e.g., `"cannot open trace file for writing"`) -- human-readable

## Message List

| ID | Symbolic Constant | Message | Severity | Trigger Condition |
|----|------------------|---------|----------|-------------------|
| 701 | `SC_ID_TRACING_FOPEN_FAILED_` | cannot open trace file for writing | ERROR | File open failure (wrong path, insufficient permissions, insufficient disk space) |
| 702 | `SC_ID_TRACING_TIMESCALE_DEFAULT_` | default timescale unit used for tracing | INFO | User did not set timescale, automatically using kernel resolution |
| 703 | `SC_ID_TRACING_TIMESCALE_UNIT_` | tracing timescale unit set | INFO | User manually set the tracing time unit |
| 704 | `SC_ID_TRACING_VCD_DELTA_CYCLE_` | VCD delta cycle tracing with pseudo timesteps (1 unit) | INFO | Delta cycle tracing enabled in VCD format |
| 705 | `SC_ID_TRACING_INVALID_TIMESCALE_UNIT_` | invalid tracing timescale unit set | ERROR | Time unit is not a power of 10 |
| 710 | `SC_ID_TRACING_OBJECT_IGNORED_` | object cannot not be traced | WARNING | Attempting to trace an unsupported type (triggers the `void*` version of `sc_trace`) |
| 711 | `SC_ID_TRACING_OBJECT_NAME_FILTERED_` | traced object name filtered | INFO | Traced object name was filtered (contains illegal characters) |
| 712 | `SC_ID_TRACING_INVALID_ENUM_VALUE_` | traced value of enumerated type undefined | WARNING | Enum value exceeds defined range |
| 713 | `SC_ID_TRACING_VCD_TIME_RESOLUTION_` | current kernel time is not representable in VCD time units | WARNING | Simulation time cannot be precisely represented in VCD time units |
| 714 | `SC_ID_TRACING_REVERSED_TIME_` | tracing cycle with duplicate or reversed time detected | WARNING | Detected time reversal or duplicate tracing timestamps |
| 715 | `SC_ID_TRACING_CLOSE_EMPTY_FILE_` | trace file closed before any cycles were traced, file not written | WARNING | Trace file closed without ever recording any data |
| 720 | `SC_ID_TRACING_ALREADY_INITIALIZED_` | sc_trace_file already initialized | ERROR | Attempting to modify timescale or add trace objects after simulation has started |

> Note: IDs 706-709 and 716-719 are currently unused, reserved for future expansion.

## SC_DEFINE_MESSAGE Macro

```cpp
#define SC_DEFINE_MESSAGE(id, unused1, unused2) \
    namespace sc_core { extern SC_API const char id[]; }
```

This macro only declares external constant strings in the header file. The actual string definitions reside in the corresponding `.cpp` file (through the mechanism in `sysc/utils/sc_report.cpp`).

`unused1` (numeric ID) and `unused2` (message text) appear to be ignored in this macro expansion, but they are used by other versions of `SC_DEFINE_MESSAGE` (when defining in `.cpp` files).

## Usage Scenarios

### Common Developer Problem Reference

| Problem You Encounter | Likely Message ID |
|-----------------------|-------------------|
| Trace file was not generated | 701 (file open failed) or 715 (no data recorded) |
| Timescale looks wrong | 702 (used default value) or 705 (invalid unit) |
| A signal does not appear in the waveform | 710 (type not supported) or 720 (added too late) |
| Timestamps in VCD file have problems | 713 (insufficient resolution) or 714 (time reversal) |

## Related Files

- [sc_trace_file_base.md](sc_trace_file_base.md) -- Primary location where these IDs are used to report errors
- [sc_trace.md](sc_trace.md) -- `SC_ID_TRACING_OBJECT_IGNORED_` is used in `sc_trace()`
- [sc_vcd_trace.md](sc_vcd_trace.md) -- VCD-specific message usage
- `sysc/utils/sc_report.h` -- Error reporting mechanism
