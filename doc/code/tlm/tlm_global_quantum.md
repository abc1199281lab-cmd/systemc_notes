# TLM Global Quantum Analysis

> **File**: `ref/systemc/src/tlm_core/tlm_2/tlm_quantum/tlm_global_quantum.h`

## 1. Overview
The `tlm_global_quantum` class manages the global time quantum for the simulation. It is a singleton.

## 2. Concept: Temporal Decoupling
In Loosely Timed (LT) modeling, initiators are allowed to run ahead of the global SystemC simulation time (`sc_time_stamp()`) to improve performance by reducing the number of context switches (calls to `wait()`).
The **Global Quantum** defines the maximum amount of time an initiator can run ahead before it *must* synchronize.

## 3. Implementation
- **Singleton**: Accessed via `instance()`.
- **`m_global_quantum`**: Stores the time duration of the quantum.
- **`compute_local_quantum()`**: Returns the global quantum. (Derived classes could potentially override this logic in more complex setups, but usually it just returns the global value).

## 4. Key Takeaways
1.  **Performance vs. Accuracy**: A larger quantum allows for faster simulation (fewer syncs) but reduces timing accuracy and interactivity between processes. A smaller quantum increases accuracy but slows down simulation.
