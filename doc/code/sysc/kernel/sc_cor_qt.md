# SystemC Coroutine (QuickThreads) Analysis

> **File**: `ref/systemc/src/sysc/kernel/sc_cor_qt.cpp`

## 1. Overview
This file implements the SystemC coroutine interface using **QuickThreads** (QT). This is typically the default and most performant backend on Unix-like systems because it performs context switching entirely in user space.

## 2. Core Mechanism

### 2.1 Stack Management
- **`stack_alloc`**: Manually allocates memory (via `mmap`) for the stack of each coroutine/thread.
- **`stack_protect`**: Uses `mprotect` to add a "red zone" at the end of the stack to detect overflows.

### 2.2 Context Switching (`yield`)
- Uses assembly helper functions (`QUICKTHREADS_BLOCK`, `sc_cor_qt_yieldhelp`) to rewrite the CPU stack pointer (`SP`) and program counter (`PC`).
- **Sanitizers**: Includes hooks (`sanitizer_start_switch_fiber`) to play nicely with AddressSanitizer (ASan), ensuring it knows about the stack switch.

## 3. Key Takeaways
1.  **Speed**: Faster than Pthreads because it avoids kernel syscalls for switching.
2.  **Fragility**: Relies on architecture-specific assembly code (in separate `.s` files usually linked in) to capture and restore register state.
