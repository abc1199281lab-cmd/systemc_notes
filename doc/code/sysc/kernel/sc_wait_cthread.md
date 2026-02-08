# SystemC Wait CThread Analysis

> **File**: `ref/systemc/src/sysc/kernel/sc_wait_cthread.cpp`

## 1. Overview
This file implements `wait()` overloads specifically for **`SC_CTHREAD`** (clocked threads). `SC_CTHREAD`s are a legacy (but still supported) construct where the thread is sensitive to a clock edge by default.

## 2. Key Functions

### 2.1 `wait(int n)`
- Waits for `n` clock cycles.
- **Mechanism**: Calls `wait_cycles(n)` on the `sc_cthread_handle`.
- **Restriction**: Can only be called from `SC_CTHREAD`. Calling it from `SC_METHOD` is an error.

### 2.2 `at_posedge` / `at_negedge`
- Utility functions to wait until a specific edge of a signal occurs.
- Implemented as a loop of `wait()` calls until the signal value matches the expected level.
- **Inefficiency**: These busy-wait loops (`do { wait(); } while (...)`) effectively skip clock cycles until the condition is met.

### 2.3 `halt()`
- Stops the execution of the `SC_CTHREAD` indefinitely.
- Internally calls `wait_halt()`.

## 3. Key Takeaways
1.  **Synthesizability**: These constructs (`wait(n)`, `at_posedge`) were originally designed for high-level synthesis (HLS) flows to express cycle-accurate behavior easily.
