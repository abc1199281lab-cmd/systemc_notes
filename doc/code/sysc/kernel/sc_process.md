# SystemC Process Analysis: Base & Method

> **Files**:
> - `ref/systemc/src/sysc/kernel/sc_process.cpp`
> - `ref/systemc/src/sysc/kernel/sc_method_process.cpp`

## 1. Overview
In SystemC, everything that "runs" is a process.
- **`sc_process_b` (Base)**: The abstract base class for all process types. It handles common functionality like state management, sensitivity, and reset.
- **`sc_method_process` (Method)**: A specific type of process that runs to completion and cannot suspend. It corresponds to combinational logic in hardware.

---

## 2. `sc_process_b` (The Base Class)
This class manages the lifecycle and state of a simulation process.

### 2.1 Process States
A process can be in multiple states (defined in `sc_process.h` but managed here):
- **`ps_normal`**: Running or ready to run.
- **`ps_bit_suspended`**: Waiting for an event/time.
- **`ps_bit_disabled`**: Disabled by user (will not run even if triggered).
- **`ps_bit_zombie`**: Marked for deletion.

### 2.2 Sensitivity Management
- **Static Sensitivity**: Events known at elaboration time (e.g., `sensitive << clk.pos()`).
    - `add_static_event(e)`: Registers the process with an event.
- **Dynamic Sensitivity**: Events waited on during simulation (e.g., `wait(e)` or `next_trigger(e)`).
    - `trigger_dynamic(e)`: Called when a dynamic event fires.

### 2.3 Reset Mechanism
It handles both synchronous and asynchronous resets (`throw_reset`).
- When a reset is asserted, it can "throw" an exception to unwind the process stack (for threads) or restart execution (for methods).

---

## 3. `sc_method_process` (SC_METHOD)
The `SC_METHOD` macro creates instances of this class.

### 3.1 Characteristics
- **No Stack**: Unlike threads, it doesn't have its own stack. It runs on the kernel's stack.
- **Run-to-Completion**: It must return control to the kernel. It cannot call `wait()`.
- **`next_trigger()`**: Instead of `wait()`, methods use `next_trigger()` to tell the kernel *when* to run them again.

### 3.2 Execution Flow (`run_process`)
Although not fully visible in the `.cpp` (it's often inline or in headers), the flow is:
1.  **Trigger**: Event fires -> `trigger_dynamic()` or `trigger_static()`.
2.  **Runnable**: Process is pushed to `sc_simcontext`'s runnable queue.
3.  **Execution**: `sc_simcontext::crunch` pops it and calls its function pointer.
4.  **Completion**: Function returns.

### 3.3 Dynamic Triggering (`trigger_dynamic`)
This function is complex because it handles different trigger types:
- `EVENT`: Single event.
- `AND_LIST`: Wait for *all* events in a list.
- `OR_LIST`: Wait for *any* event in a list.
- `TIMEOUT`: Wait for time.
- `EVENT_TIMEOUT`: Wait for event OR time.

When the condition is met, it checks if the process is suspended/disabled. If valid, it pushes the process to the runnable queue:
```cpp
simcontext()->push_runnable_method(this);
```

---

## 4. Hardware / RTL Mapping

| SystemC (`sc_method_process`) | Verilog / SystemVerilog |
| :--- | :--- |
| `SC_METHOD` | `always @(sensitivity)` or `assign` |
| `next_trigger(e)` | Changing the `@(sensitivity)` list dynamically (rare in RTL, common in testbenches) |
| `dont_initialize()` | `always @(posedge clk)` (Wait for first edge), vs `initial begin ... forever ... end` |

- **Combinational Logic**: `SC_METHOD` is the primary way to model `always @(*)` or `assign` statements.
- **Sequential Logic (Synchronous)**: Can also be modeled if sensitive to a clock edge, but `SC_CTHREAD` is often preferred for synthesis-like style.

---

## 5. Key Takeaways
1.  **Efficiency**: `sc_method_process` is much lighter than `sc_thread_process` because it avoids context switching overhead. Use it whenever possible.
2.  **Stateless execution**: Since it returns, local variables are lost. State must be stored in class members (modules).
3.  **Exception Handling**: It has robust mechanisms (`throw_user`, `throw_reset`) to handle errors and resets, even though it doesn't have a persistent stack.
