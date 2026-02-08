# Top-Down Analysis: Process Scheduling & Concurrency

Concurrency in SystemC is **simulated** (unless using modern parallel extensions). The simulation kernel gives the illusion of parallel hardware executing by rapidly switching between processes.

## 1. The Process Types
SystemC provides two fundamental process types, each serving a specific modeling need.

### 1.1 Method Processes (`SC_METHOD`)
-   **Abstration**: Combinational Logic / Reactive Software.
-   **Mechanism**: A standard C++ member function.
-   **Stack**: Runs on the kernel's stack (or the stack of the thread triggering it).
-   **Behavior**:
    -   Must run to completion.
    -   **Cannot** call `wait()`.
    -   Retains no local state between invocations (unless stored in class members).
-   **Use Case**: Implementing RTL-like boolean logic, simple arithmetic, or handling interrupts.

### 1.2 Thread Processes (`SC_THREAD`)
-   **Abstraction**: Sequential Logic / Testbenches / Software Tasks.
-   **Mechanism**: A **Coroutine** (User-Level Thread).
-   **Stack**: Allocated a private stack (e.g., 64KB).
-   **Behavior**:
    -   Can suspend execution using `wait()`.
    -   Preserves local variable state across suspensions.
    -   Runs essentially "forever" (usually an infinite `while(true)` loop).
-   **Use Case**: Writing test sequences (e.g., "drive A, wait 10ns, drive B"), complex state machines, or high-level algorithmic models.

### 1.3 Clocked Threads (`SC_CTHREAD`)
-   **Abstraction**: Synthesizable Synchronous Logic.
-   **Mechanism**: A specialized `SC_THREAD` that is sensitive *only* to a clock edge.
-   **Legacy**: largely superseded by `SC_THREAD` or `SC_METHOD` in modern usage, but historically important for High-Level Synthesis (HLS).

## 2. The Scheduler's View
The scheduler (`sc_simcontext`) doesn't "know" what the user code does. It simply sees `sc_process_b` handles.

### 2.1 The Runnable Queue
-   The scheduler maintains a list (or queue) of processes that are ready to run in the current delta cycle.
-   **Non-Preemptive**: Once a process starts (Method or Thread), the scheduler **cannot** interrupt it. The process must voluntarily yield (return for Method, `wait()` for Thread).
-   **Order**: The execution order of processes within the same delta cycle is **undefined** by the standard. This enforces the discipline of using signals (Update phase) for communication to avoid race conditions.

### 2.2 Context Switching (Threads)
When `sc_simcontext` picks an `SC_THREAD` to run:
1.  **Save Kernel**: It saves the current CPU registers (Instruction Pointer, Stack Pointer) to the kernel's context.
2.  **Restore Thread**: It loads the saved registers from the thread's context block.
3.  **Resume**: The CPU jumps to the instruction after the last `wait()` call inside the user's thread.
4.  **Suspend**: When the user calls `wait()`, the reverse happens.

This context switching is implemented via the "Coroutine Package" (`sc_cor_pkg`), which wraps underlying mechanisms like:
-   `setjmp`/`longjmp` (QuickThreads)
-   Windows Fibers
-   Pthreads (for standard C++ implementation)

## 3. Key Files
-   `sysc/kernel/sc_process.cpp`: The base class managing process state (suspended, normal, zombie).
-   `sysc/kernel/sc_method_process.cpp`: Implementation of the lightweight method runner.
-   `sysc/kernel/sc_thread_process.cpp`: Implementation of the stack-based thread.
-   `sysc/kernel/sc_cor.h`: The interface for the low-level context switchers.
