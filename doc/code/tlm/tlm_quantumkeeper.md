# TLM Quantum Keeper Analysis

> **File**: `ref/systemc/src/tlm_utils/tlm_quantumkeeper.h`

## 1. Overview
`tlm_quantumkeeper` is a utility class provided in `tlm_utils` to help initiators implement temporal decoupling.

## 2. State Variables
- **`m_local_time`**: The time offset by which the initiator is currently ahead of `sc_time_stamp()`.
- **`m_next_sync_point`**: The absolute SystemC time when the next synchronization is required (`sc_time_stamp() + quantum`).

## 3. Mechanism
- **`inc(t)`**: Accumulates local time delay (e.g., adding transport delays, computation delays).
- **`need_sync()`**: Checks if the accumulated `m_local_time` plus current time has reached or exceeded `m_next_sync_point`.
- **`sync()`**:
    1.  Calls `sc_core::wait(m_local_time)` to actually advance SystemC time.
    2.  Resets `m_local_time` to zero.
    3.  Calculates the new `m_next_sync_point`.

## 4. Usage Pattern
1.  Initiator does work (reads/writes).
2.  Initiator calls `keeper.inc(delay)`.
3.  Initiator checks `if (keeper.need_sync()) keeper.sync();`.

## 5. Key Takeaways
1.  **Standard Utility**: While users can implement their own decoupling logic, `tlm_quantumkeeper` provides a standardized, robust way to do it.
2.  **namespace**: Note that this class is in `tlm_utils`, not `tlm`.
