# SystemC Tracing: Trace File Base (`sc_trace_file_base`)

> **Source**: `ref/systemc/src/sysc/tracing/sc_trace_file_base.cpp`

## 1. Overview
`sc_trace_file_base` is the common base class for specific trace file implementations (like VCD or WIF). It handles the administrative tasks of tracing that are format-independent.

## 2. Key Responsibilities

### 2.1 File Management
-   Manages the file pointer (`fp`) and filename.
-   Handles opening (`open_fp`) and closing (`fclose`) the file.

### 2.2 Time Management
-   **Units**: Stores the relationship between the kernel's time resolution and the trace file's time unit.
-   **Conversion**: `unit_to_fs` and `fs_unit_to_str` convert between SystemC time enums and strings/integers.
-   **Scaling**: Calculates the divisor needed if valid trace units differ from kernel units.

### 2.3 Simulation Phase Callbacks
-   Registers with the simulation kernel via `sc_register_stage_callback`.
-   **Triggering**: The `stage_callback` method is called by the kernel.
    -   It triggers a trace cycle at `SC_POST_UPDATE` if delta cycle tracing is enabled.

## 3. Initialization
-   The `initialize()` method ensures the file is open and the timescale is written (via `do_initialize`, which derived classes implement) before the first write occurs.

## 4. Key Takeaways
-   Centralizes logic for "when to dump" and "where to dump".
-   Abstracts the "how to format" to derived classes.
-   Delta cycle tracing logic lives here.
