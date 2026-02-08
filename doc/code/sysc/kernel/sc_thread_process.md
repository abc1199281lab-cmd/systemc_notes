# SystemC Process Analysis: Thread & CThread

> **Files**:
> - `ref/systemc/src/sysc/kernel/sc_thread_process.cpp`
> - `ref/systemc/src/sysc/kernel/sc_cthread_process.cpp`

## 1. Overview
Threads in SystemC are processes that **have their own stack** and **can suspend** execution (preserve state across wait calls).
- **`sc_thread_process`**: The standard `SC_THREAD`.
- **`sc_cthread_process`**: A legacy/specialized `SC_CTHREAD` (Clocked Thread), mainly for High-Level Synthesis (HLS) constraints.

---

## 2. `sc_thread_process` (SC_THREAD)
This is the workhorse of behavioral modeling.

### 2.1 The Coroutine Mechanism
Unlike `SC_METHOD`, a thread uses a coroutine.
- `prepare_for_simulation()`: Calls `simcontext()->cor_pkg()->create(...)` to allocate a separate stack for this thread.
- `sc_thread_cor_fn`: The static entry wrapper function. It catches exceptions and calls `thread_h->semantics()` (which eventually calls the user's function).

### 2.2 Suspend & Resume
- **`suspend_process()`**:
    - Marks state as `ps_bit_suspended`.
    - Context switches away from this thread efficiently.
- **`wait()` (Implicit)**: When a user calls `wait(e)`, the kernel calls `suspend_process()` behind the scenes and registers the sensitivty.

### 2.3 Exception Handling & Stack Safety
Since threads have stacks, throwing exceptions across the kernel boundary is tricky.
- **`throw_user`**: Allows one process to throw an exception *into* another process.
- **`throw_reset`**: Asynchronous resets are implemented by injecting an exception into the thread to force stack unwinding (or just jumping if using `setjmp/longjmp` based coroutines).
- **`SC_ALIGNED_STACK_`**: The code includes architecture-specific macros (x86, x64, Cygwin) to ensure the stack is correctly aligned (16-byte) for modern compilers/CPUs.

---

## 3. `sc_cthread_process` (SC_CTHREAD)
A "Clocked Thread" is a specialized version of a thread.

### 3.1 Key Differences
- **Inheritance**: It inherits from `sc_thread_process`.
- **Static Sensitivity**: It is *only* sensitive to the clock edge defined at construction.
- **`dont_initialize`**: It calls `m_dont_init = true` in the constructor. `SC_CTHREAD`s typically do not run at time 0 (initialization phase), waiting for the first clock edge instead.

### 3.2 Legacy Status
- While widely used in early SystemC for synthesis, modern HLS tools often accept `SC_THREAD` with specific coding styles (infinite loop + wait at start).
- However, `SC_CTHREAD` enforcement of "wait only on clock" helped early synthesis tools.

---

## 4. Hardware / RTL Mapping

| SystemC (`sc_thread_process`) | Verilog / SystemVerilog |
| :--- | :--- |
| `SC_THREAD` | `initial begin ... end` or `always` block with delays |
| `wait(10, SC_NS)` | `#10;` |
| `wait(event)` | `@(event);` |
| `sc_cthread_process` | `always @(posedge clk)` |
| `wait()` (in CTHREAD) | Wait for next active clock edge |

- **Behavioral Modeling**: Threads are ideal for writing testbenches (stimulus generation) where you want linear, sequential code flow.
- **HLS**: Used to describe complex sequential algorithms that span multiple clock cycles.

---

## 5. Key Takeaways
1.  **Context Switching Cost**: `SC_THREAD` is heavier than `SC_METHOD` because swapping stacks takes CPU time. Use Method for pure combinational logic.
2.  **Stack Size**: Each thread consumes memory. `SC_DEFAULT_STACK_SIZE` (often 64KB or 128KB) can be overridden if you have thousands of threads or deep recursion.
3.  **Simulation Speed**: Too many threads can slow down simulation due to cache pollution from stack swapping.
