# SystemC Stage Callback Registry Analysis

> **File**: `ref/systemc/src/sysc/kernel/sc_stage_callback_registry.cpp`

## 1. Overview
Implements the registry for **simulation stage callbacks**. This modern SystemC feature allows users and tools (debuggers, tracers, inspectors) to register functions to be called at specific points in the simulation loop.

## 2. Stages (`sc_stage`)
Defined in `sc_stage_callback_if.h`, these bitmasks represent execution phases:
- `SC_POST_END_OF_ELABORATION`
- `SC_POST_UPDATE` (after request_update/update phase)
- `SC_PRE_TIMESTEP` (before time advances)
- `SC_POST_END_OF_SIMULATION`
- ... and others.

## 3. Registry Mechanism
- **registration**: `register_callback(cb, mask)` adds a callback for specific stages.
- **optimization**: Maintains separate vectors (`m_cb_update_vec`, `m_cb_timestep_vec`) for high-freq stages like UPDATE and TIMESTEP to avoid iterating over the full list during the critical path of the simulation loop.
- **validation**: Checks if the requested mask is valid and if the stage has already passed (e.g., trying to register for Elaboration Done after simulation started).

## 4. Key Takeaways
1.  **Instrumentation**: This is the primary hook for building non-intrusive Simulation control and inspection tools.
