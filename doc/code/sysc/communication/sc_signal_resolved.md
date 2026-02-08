# SystemC Resolved Signal Analysis

> **File**: `ref/systemc/src/sysc/communication/sc_signal_resolved.cpp`

## 1. Overview
`sc_signal_resolved` is a specialized channel for `sc_logic` that supports **multiple writers** (bus modeling). It resolves conflicting values (e.g., '0' and '1' driven simultaneously) using a resolution table.

## 2. Resolution Mechanism

### 2.1 Resolution Table (`sc_logic_resolution_tbl`)
A 4x4 table defining the result of driving two `sc_logic` values together:
- `0` + `0` -> `0`
- `0` + `1` -> `X` (Conflict)
- `0` + `Z` -> `0` (Dominant)
- `1` + `1` -> `1`
- ...

### 2.2 Tracking Drivers
- **`write(val)`**: Unlike `sc_signal` which stores a single new value, `sc_signal_resolved` maintains a list of values (`m_val_vec`), one for each write process (`m_proc_vec`).
- This allows it to remember what *each* process is driving during the delta cycle.

### 2.3 Update
- **`update()`**: Calls `sc_logic_resolve` to reduce the vector of driven values into a single final value using the resolution table.

## 3. Key Takeaways
1.  **Wire-AND/OR Modeling**: Essential for modeling tri-state buses or bidirectional lines where multiple components might drive the line simultaneously.
