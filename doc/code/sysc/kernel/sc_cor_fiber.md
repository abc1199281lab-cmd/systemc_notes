# SystemC Coroutine (Windows Fibers) Analysis

> **File**: `ref/systemc/src/sysc/kernel/sc_cor_fiber.cpp`

## 1. Overview
This file implements SystemC coroutines using **Windows Fibers**. Fibers are a lightweight thread-like primitive provided by the Windows API specifically for cooperative multitasking.

## 2. Core Mechanism

### 2.1 Main Thread Conversion
- **`ConvertThreadToFiber`**: The main simulation thread must first be converted into a Fiber before it can schedule other Fibers.

### 2.2 Creation and Switching
- **`CreateFiberEx`**: Creates the coroutine stack management structure managed by the OS.
- **`SwitchToFiber`**: The actual context switch call. It saves the current state and restores the target fiber state.

## 3. Key Takeaways
1.  **OS Support**: This is the native way to do user-space threads on Windows.
2.  **SJLJ Exceptions**: On some GCC/MinGW setups, it includes special handling (`_Unwind_SjLj_Register`) to ensure C++ exceptions propagate correctly across fiber switches.
