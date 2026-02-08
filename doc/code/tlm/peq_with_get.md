# TLM PEQ with Get Analysis

> **File**: `ref/systemc/src/tlm_utils/peq_with_get.h`

## 1. Overview
`peq_with_get` (Payload Event Queue with Get) is a utility class in the `tlm_utils` namespace. It provides a mechanism to delay the processing of transactions until a specific time, allowing processes to "wait" for transactions.

## 2. Mechanism
- **Event Based**: It has an internal `sc_event m_event` that is notified when a transaction becomes available (due to time advancing).
- **`notify(trans, t)`**: Inserts a transaction into the queue scheduled for time `t` (relative to `sc_time_stamp()`). It calls `m_event.notify(t)` to wake up the consumer at the right time.
- **`get_next_transaction()`**:
    - Checks if there are any events scheduled for the current simulation time (or earlier).
    - If yes, returns the transaction and removes it from the queue.
    - If no, returns `NULL`.
    - If there are future events, it re-notifies `m_event` for the delta time to the next event.

## 3. Usage Pattern
This is typically used in `SC_THREAD` processes in models that need to accept non-blocking calls but process them in a blocking/sequential manner.
```cpp
// In target's nb_transport_fw
peq.notify(trans, delay);
return TLM_ACCEPTED;

// In target's SC_THREAD
void process_thread() {
  while(true) {
    wait(peq.get_event());
    while ((trans = peq.get_next_transaction()) != NULL) {
      // process trans
    }
  }
}
```

## 4. Key Takeaways
1.  **Bridge**: Acts as a bridge between the non-blocking realm (incoming `nb_transport`) and the blocking realm (worker thread).
