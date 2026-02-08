# SystemC SimContext Analysis

> **File**: `ref/systemc/src/sysc/kernel/sc_simcontext.cpp`

## 1. Overview
`sc_simcontext` is the **central nervous system** of the SystemC simulation kernel. It is a singleton-like object (though multiple instances can technically exist) that manages:
1.  **Global Simulation State**: Time, Delta cycles, Simulation status (elaboration, running, paused).
2.  **Object Registry**: Tracks all modules, ports, channels, and processes.
3.  **Scheduler**: The `crunch()` method is the heart of the discrete-event simulation algorithm.
4.  **Process Management**: Holds the `m_process_table` for methods and threads.

---

## 2. Core Components

### 2.1 The Scheduler Loop: `crunch()`
This is the most critical function in the entire kernel. It implements the **Evaluate-Update-Notify** delta cycle loop.

```cpp
// Simplified Logic of crunch()
void sc_simcontext::crunch( bool once ) {
    while (true) {
        // 1. EVALUATE PHASE
        // Run all runnable processes (methods and threads)
        // This is where user code executes!
        
        while (runnable_processes_exist) {
            execute_method_processes();
            execute_thread_processes();
        }

        // 2. UPDATE PHASE
        // Primitive channels (like sc_signal) update their values here
        // to ensure improved determinism (avoid race conditions).
        m_prim_channel_registry->perform_update();

        // 3. NOTIFICATION PHASE
        // Trigger events that were notified during evaluate/update
        // This may make new processes runnable for the NEXT delta cycle.
        trigger_delta_events();

        // 4. Time Advancement Check
        // If no more runnable processes and no more delta events,
        // the delta cycle loop breaks, and time advances in `sc_start`.
    }
}
```

### 2.2 Process Management
It uses `sc_process_table` to keep track of:
- **Method Processes** (`SC_METHOD`): Run to completion, cannot suspend.
- **Thread Processes** (`SC_THREAD`): Can suspend (`wait()`), requires coroutine context switching.

### 2.3 Object Registries
`sc_simcontext` owns several registries to manage the hierarchy:
- `m_object_manager`: General object management.
- `m_module_registry`: Tracks all `sc_module` instances.
- `m_port_registry` / `m_export_registry`: Tracks connectivity.

---

## 3. Hardware / RTL Mapping

| SystemC (`sc_simcontext`) | Hardware / Verilog Simulator |
| :--- | :--- |
| `sc_simcontext` | The **Simulator Kernel** itself (the engine). |
| `crunch()` Loop | The **Event-Driven Simulation Algorithm**. |
| `m_delta_count` | The **Delta Cycle** counter (virtual time steps). |
| `m_curr_time` | **$time** (Simulation Time). |
| `phase_evaluate` | **Active Region** (Executing `always` blocks). |
| `phase_update` | **NBA Region** (Non-Blocking Assignment updates `<=`). |

### 3.1 The "Delta Cycle" Concept
In Verilog:
```verilog
always @(posedge clk) begin
    a <= b; // NBA
    c <= a; // Reads OLD value of a
end
```
In SystemC, `sc_simcontext` enforces this behavior via the **Evaluate-Update** phases.
1.  **Evaluate**: `a.write(b)` is called. The new value is *staged* but not applied.
2.  **Update**: `sc_simcontext` calls `perform_update()`, applying the new value to `a`.
This ensures `c` reads the old value of `a` if it runs in the same evaluation phase.

---

## 4. Key Takeaways
1.  **Determinism**: The scheduler's separation of Evaluate and Update phases is what makes SystemC cycle-accurate and deterministic, matching hardware behavior.
2.  **Centralization**: Almost everything in SystemC (creating a module, creating a signal, notifying an event) eventually calls back into `sc_simcontext` to register itself.
3.  **Coroutines**: It manages the coroutine package (`m_cor_pkg`) to support `SC_THREAD` context switching, which is the software implementation of "concurrent hardware processes".
