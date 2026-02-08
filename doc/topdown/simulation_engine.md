# Top-Down Analysis: The Simulation Engine

SystemC is effectively a **co-operative multitasking OS** specialized for hardware simulation. At its core lies the `sc_simcontext`, which acts as the kernel, scheduler, and event manager. This document breaks down how the simulation engine drives time and events.

## 1. The Startup Sequence
Before any simulation can run, the "world" must be built. This is the **Elaboration Phase**.

1.  **Entry Point**: User writes `sc_main`. The library provides the C++ `main`, which calls `sc_elab_and_sim`.
2.  **Construction**: Global constructors execute. Modules (`sc_module`) and signals (`sc_signal`) register themselves with `sc_simcontext::m_module_registry` and `m_object_manager`.
3.  **Elaboration**: `sc_start()` is called. The kernel calls `before_end_of_elaboration` and `end_of_elaboration` callbacks on all modules, finalizing the hierarchy and port binding.

## 2. The Simulation Loop (The "Crunch")
The heart of SystemC is the **Evaluate-Update-Notify** loop, implemented in `sc_simcontext::crunch()`. This loop ensures deterministic behavior similar to Verilog's delta cycles.

### 2.1 Phase 1: Evaluation (The "Run" Phase)
-   **Action**: The kernel iterates through the list of **runnable processes**.
-   **Execution**:
    -   `SC_METHOD`: The C++ method is called directly. It runs to completion and returns.
    -   `SC_THREAD`: The kernel switches the CPU context (stack pointer) to the thread's stack. The thread runs until it calls `wait()`, which switches context back to the kernel.
-   **Result**: Processes read current signal values and request new values (e.g., `sig.write(val)`). Crucially, **primitive channels do not update immediately**. They store the new value in a temporary buffer.

### 2.2 Phase 2: Update (The "Commit" Phase)
-   **Action**: Once the runnable queue is empty, the kernel calls `perform_update()` on the `prim_channel_registry`.
-   **Execution**: All primitive channels (Signals) with pending values apply them now.
-   **Significance**: This guarantees that order of process execution within the Evaluation phase doesn't change the values read by other processes. Everyone sees the "old" value until the Update phase.

### 2.3 Phase 3: Notification (The "Trigger" Phase)
-   **Action**: The kernel checks for generated events (`sc_event::notify()`).
-   **Delta Notification**: If a signal changed during Update, it notifies sensitive processes.
-   **Timed Notification**: The kernel checks the generic event list.
-   **Result**: If new processes are triggered (made runnable), the loop jumps back to **Phase 1** without advancing simulation time. This is a **Delta Cycle**.

### 2.4 Time Advancement
-   If no processes are runnable and no delta events are pending, the engine looks for the **nearest future timed event**.
-   **Action**: Simulation time (`sc_time_stamp`) is advanced to that timestamp. The waiting processes are made runnable, and the loop restarts.

## 3. Hardware Mapping
| SystemC Phase | Verilog Equivalent | concept |
| :--- | :--- | :--- |
| **Evaluate** | Active Region | `always` blocks executing |
| **Update** | NBA Region | Non-Blocking Assignment (`<=`) updates |
| **Delta Cycle** | Delta Delay | Virtual time steps at same simulation time |
| **Time Advance** | `#delay` | Moving forward in simulation time |

## 4. Key Files
-   `sysc/kernel/sc_simcontext.cpp`: Implements the `crunch()` loop and time management.
-   `sysc/kernel/sc_main.cpp`: The bridge that bootstraps the kernel.
-   `sysc/communication/sc_prim_channel.cpp`: Implements the `update()` request logic.
