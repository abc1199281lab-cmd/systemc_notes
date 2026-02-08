# TLM LT Temporal Decoupling Example Analysis

> **Source**: `ref/systemc/examples/tlm/lt_temporal_decouple` & `ref/systemc/examples/tlm/common`

## 1. Overview
This example demonstrates **Temporal Decoupling** in the Loosely Timed (LT) coding style. Initiators are allowed to run ahead of the simulation time by a "Quantum", accumulating local delays instead of calling `wait()` immediately. This significantly improves simulation performance by reducing context switching.

## 2. Key Components

### 2.1 Quantum Keeper (`tlm_utils::tlm_quantumkeeper`)
-   **Role**: Manages the local time offset for the initiator.
-   **Global Quantum**: A global setting (e.g., 500ns) that determines how far ahead an initiator can run.

### 2.2 TD Initiator (`lt_td_initiator.cpp`)
-   **Loop**:
    1.  **Get Offset**: `m_delay = m_quantum_keeper.get_local_time()`.
    2.  **Transact**: Calls `b_transport(trans, m_delay)`. The target adds its processing time to `m_delay`.
    3.  **Update Keeper**: `m_quantum_keeper.set(m_delay)`. The keeper now knows the new local time.
    4.  **Check Sync**: `if (m_quantum_keeper.need_sync()) m_quantum_keeper.sync()`.
        -   If the accumulated delay > Global Quantum, the keeper calls `wait(delay)` and resets local time to 0.

### 2.3 Synch Target (`lt_synch_target.cpp`)
A "misbehaving" or "legacy" target that requires synchronization.
-   **Behavior**: Instead of just adding to `delay_time`, it explicitly calls `wait(delay_time)`.
-   **Impact**: It forces the simulation time to advance.
-   **Reset**: It acts as a synchronization point. It sets `delay_time = SC_ZERO_TIME` before returning.
-   **Initiator Handling**: The initiator sees `delay` as 0. `m_quantum_keeper.set(0)` correctly resets the local offset because the target has already "paid" the time cost.

## 3. Execution Flow
1.  **Start**: Global Quantum = 500ns.
2.  **Transaction 1**: Target adds 100ns. `m_delay` = 100ns. No Sync.
3.  **Transaction 2**: Target adds 100ns. `m_delay` = 200ns. No Sync.
...
4.  **Transaction 6**: Target adds 100ns. `m_delay` = 600ns.
5.  **Sync**: 600ns > 500ns. Initiator calls `wait(600ns)`. Simulation time advances. `m_delay` resets to 0.

## 4. Key Takeaways
-   **Speed**: Drastically reduces the number of `wait()` calls and context switches.
-   **Trade-off**: Timing accuracy is reduced. Events happening within the same quantum effectively happen "at the same time" from the perspective of other processes.
-   **Best Practice**: Use Temporal Decoupling for instruction set simulators (ISS) and high-speed functional models.
