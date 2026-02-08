# TLM PEQ with Callback and Phase Analysis

> **File**: `ref/systemc/src/tlm_utils/peq_with_cb_and_phase.h`

## 1. Overview
`peq_with_cb_and_phase` is a utility class in `tlm_utils` designed for fully non-blocking models (AT style). Instead of waiting on an event, it triggers a **callback method** when a transaction becomes ripe.

## 2. Mechanism
- **Callback**: Takes a function pointer (method) in the constructor: `peq(this, &MyModule::my_callback)`.
- **Internal Queues**:
    - `m_immediate_yield`: For immediate notifications.
    - `m_even_delta` / `m_uneven_delta`: For delta-cycle delays (handling the specific SystemC delta cycle logic).
    - `m_ppq` (Payload Priority Queue): For timed notifications.
- **`notify(trans, phase, delay)`**: Schedules the call.
- **`fec()` (Fire Event Call)**: An internal `SC_METHOD` sensitive to the internal event. When triggered, it checks the queues and calls the registered callback for all ripe transactions.

## 3. Usage Pattern
Used in AT Initiators and Targets to model delays without blocking threads.
```cpp
// Incoming path
peq.notify(trans, phase, delay);

// Callback
void my_callback(trans, phase) {
  // Logic to handle phase transition, e.g., send END_REQ or BEGIN_RESP
}
```

## 4. Key Takeaways
1.  **AT Modeling**: Essential for implementing the complicated state machines of Approximately Timed components where multiple transactions can be in flight.
2.  **Centralized Processing**: Funnels all time-delayed events into a single callback method, simplifying state management.
