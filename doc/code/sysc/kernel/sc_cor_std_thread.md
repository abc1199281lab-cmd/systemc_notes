# SystemC Coroutine (std::thread) Analysis

> **File**: `ref/systemc/src/sysc/kernel/sc_cor_std_thread.cpp`

## 1. Overview
A modern implementation of SystemC coroutines using C++11 `std::thread`. This is similar to the Pthreads implementation but uses the standard C++ API, making it more portable without external dependencies.

## 2. Core Mechanism

### 2.1 Handshake Logic
Just like the Pthreads version, it uses a mutex/condition variable pair (`m_create_mutex`, `m_create_cond`) to perform the handshake.
1.  **Main**: Creates `std::thread`, waits.
2.  **Child**: Starts, notifies Main, then waits on its own condition variable.
3.  **Main**: Wakes up, proceeds.

### 2.2 Unlocking
It uses `std::unique_lock` and `std::defer_lock` to manage the delicate locking sequence required to yield control safely between threads.

## 3. Key Takeaways
1.  **Modern Alternative**: Cleaner than raw Pthreads but suffers from the same performance overhead (OS-level switching).
2.  **Robustness**: Preferable for general debugging or when QuickThreads is not available for the target architecture.
