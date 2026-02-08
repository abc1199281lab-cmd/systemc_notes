# SystemC Exception Analysis

> **File**: `ref/systemc/src/sysc/kernel/sc_except.cpp`

## 1. Overview
This file handles the translation and processing of exceptions within the simulation kernel. It specifically defines the `sc_unwind_exception` which is used to implement `sc_thread` termination (reset or kill).

## 2. Unwinding (`sc_unwind_exception`)
This is a special exception thrown by the kernel into a thread to force it to terminate or reset.
- **`is_reset`**: Distinguishes between a "Kill" (terminate) and a "Reset" (restart).
- **`what()`**: Returns "KILL" or "RESET".
- **Safety**: The destructor checks if it is being destroyed while still active (e.g., if the user tries to swallow it). If so, it issues a fatal error because unwinding must not be blocked.

## 3. Global Exception Handler (`sc_handle_exception`)
A global catch-all function used by `sc_main` and the kernel's process dispatcher to catch exceptions escaping from user code.
- Catches `sc_report`
- Catches `std::exception`
- Catches unknown exceptions (`...`)
- Wraps them all into a dynamically allocated `sc_report*` for uniform handling.

## 4. Key Takeaways
1.  **Don't Swallow Unwind**: User code should generally not catch `sc_unwind_exception` (or `...`) without re-throwing, otherwise, the thread/process cannot correctly shut down or reset.
