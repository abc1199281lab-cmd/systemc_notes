# SystemC Signal Analysis

> **File**: `ref/systemc/src/sysc/communication/sc_signal.cpp`

## 1. Overview
`sc_signal<T>` is the fundamental primitive channel for point-to-point communication.

## 2. Implementation Details

### 2.1 Events (`sc_signal_channel`)
- Uses **Lazy Event Allocation**: Events (`m_change_event_p`, `m_posedge_event_p`) are only allocated if they are requested (e.g., by a process waiting on them). This saves memory for signals that are just read but never waited upon.

### 2.2 Update Logic (`do_update`)
- Called by `sc_prim_channel::perform_update` at the end of delta.
- Checks if `m_new_val != m_cur_val`.
- If changed:
    1.  Updates `m_cur_val`.
    2.  Notifies `value_changed_event`.
    3.  Notifies `posedge_event` or `negedge_event` (for bool/logic signals).

### 2.3 Writer Policies
- Checks for multiple drivers if `SC_ONE_WRITER` policy is active.
- `sc_signal_invalid_writer` reports conflicts.

## 3. Key Takeaways
1.  **Optimization**: The lazy event mechanism is a significant memory optimization.
2.  **Compliance**: strictly enforces single-driver rules unless configured otherwise.
