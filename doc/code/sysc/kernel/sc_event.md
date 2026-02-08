# SystemC Event Analysis

> **File**: `ref/systemc/src/sysc/kernel/sc_event.cpp`

## 1. Overview
`sc_event` is the fundamental synchronization primitive in SystemC. It allows processes to suspend execution and resume when a specific condition occurs. Unlike signals, events have no value and no duration; they are instantaneous.

## 2. Core Components

### 2.1 The `sc_event` Class
- **Purpose**: Represents a specific occurrence in time.
- **State**: Contains lists of processes waiting on it:
    - `m_methods_static` / `m_methods_dynamic`
    - `m_threads_static` / `m_threads_dynamic`

### 2.2 Notification Mechanism (`notify`)
The `notify()` method is how an event is triggered. It interacts heavily with `sc_simcontext`.

1.  **Immediate Notification** (`notify()` no args):
    - Causes `trigger()` to happen *immediately* (if in evaluation phase).
    - **Warning**: Can be non-deterministic if used carelessly.
2.  **Delta Notification** (`notify(SC_ZERO_TIME)`):
    - Adds the event to `sc_simcontext`'s delta event queue.
    - It will fire at the start of the *next* delta cycle (Update/Notify phase).
3.  **Timed Notification** (`notify(time)`):
    - Adds to the timed event queue.
    - Fires when simulation time advances.

### 2.3 The Triggering Process (`trigger`)
When an event fires (is pulled from the queue by `sc_simcontext`), `trigger()` is called:
1.  **Iterates** through all static and dynamic lists of waiting processes.
2.  **Activates**:
    - For `SC_METHOD`: Calls `trigger_static` or `trigger_dynamic` (pushes to runnable queue).
    - For `SC_THREAD`: Resumes execution after the `wait()`.
3.  **Cleanups**: Dynamic lists are cleared after triggering (since `wait(e)` is a one-time sensitivity).

### 2.4 Event Lists (`sc_event_list`)
Handling interactions like `wait(e1 | e2)` or `wait(e1 & e2)`.
- **`OR_LIST`**: The first event to fire resumes the process and removes the process from the other events' lists.
- **`AND_LIST`**: The process maintains a counter. It only resumes when *all* events have fired (typically used with `SC_JOIN`).

---

## 3. Hardware / RTL Mapping

| SystemC (`sc_event`) | Verilog / SystemVerilog |
| :--- | :--- |
| `sc_event e;` | `event e;` |
| `e.notify()` | `-> e;` |
| `wait(e)` | `@(e);` |
| `wait(e1 | e2)` | `@(e1 or e2);` |
| `notify(SC_ZERO_TIME)` | Non-blocking assignment logic (sort of), or just `@(event)` executing in next NBA region. |

- **Concept**: Events are the underlying engine for `sc_signal`. When a signal changes value, it effectively calls `notify()` on an internal event (`default_event()`), which wakes up sensitivity lists.

---

## 4. Key Takeaways
1.  **No Duration**: An event is a point in time. If you `wait(e)` *after* `e.notify()` has happened, you miss it.
2.  **Memory Management**: `sc_event` cleanup is handled delicately, especially when processes die or events are part of an event list.
3.  **Kernel Event**: There are special "kernel events" (prefixed) used for internal synchronization that don't show up in the object hierarchy.
