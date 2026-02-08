# SystemC Coroutine (Pthreads) Analysis

> **File**: `ref/systemc/src/sysc/kernel/sc_cor_pthread.cpp`

## 1. Overview
This file implements the SystemC coroutine interface (`sc_cor` and `sc_cor_pkg`) using **POSIX Threads** (`pthreads`). This is one of the standard portable backends for SystemC process switching.

## 2. Core Mechanism

### 2.1 Thread Creation
- **`create(...)`**: Spawns a new real OS thread (`pthread_create`).
- **Blocking**: The main thread immediately blocks on a condition variable (`m_create_cond`) waiting for the child to start.
- **Child Handshake**: The child thread starts, signals the main thread "I'm alive", and then immediately blocks on its own condition variable (`m_pt_condition`), waiting to be "switched in".
- **Result**: You have a pool of OS threads, but only one runs at a time (cooperative multitasking).

### 2.2 Context Switching (`yield`)
- **`yield(next_cor)`**: To switch from `A` to `B`:
    1.  `A` signals `B`'s condition variable.
    2.  `A` waits on `A`'s condition variable.
    3.  `B` wakes up and resumes execution.

## 3. Key Takeaways
1.  **Strict Sequentiality**: Even though it uses OS threads, SystemC enforces strict serialization. Only one thread holds the "active" token at a time.
2.  **Performance**: Pthreads are generally heavier than user-mode switching (QuickThreads or Fibers) because context switching involves the OS kernel scheduler.
