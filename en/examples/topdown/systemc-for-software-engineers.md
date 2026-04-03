# SystemC -- A Global Perspective for Software Engineers

> This article explains SystemC core concepts using concepts familiar to software engineers.
> You do not need any hardware background to understand this document.

---

## What is SystemC?

**In one sentence**: SystemC is a C++ class library that lets you model hardware systems by writing C++ programs.

It is not a new language, but a set of classes and macros defined in `systemc.h`. You compile with a standard C++ compiler (gcc / clang), link against the SystemC library, and then run it like a normal program.

```
Your SystemC model (.cpp)
        |
        v
  Standard C++ Compiler (g++)
        |
        v
  Link with libsystemc.a
        |
        v
  Executable (simulator)
```

### Why Should Software Engineers Care?

1. **HW/SW Co-design**: In SoC development, firmware and hardware need to be developed simultaneously. SystemC lets you test firmware before the hardware is built.

2. **Embedded system verification**: Your driver code can run in a SystemC simulation environment without waiting for actual silicon.

3. **Performance modeling**: Use SystemC to simulate system architecture and estimate performance bottlenecks before tape-out.

4. **Understanding hardware behavior**: Even if you don't design hardware, understanding how hardware works helps you write better low-level software.

---

## Core Concept Mapping Table

The following maps SystemC core concepts to their software equivalents:

| SystemC Concept | Software Equivalent | Description |
|----------------|--------------------|----|
| `sc_module` | class / component | Reusable hardware module |
| `sc_port` | dependency injection interface | External connection point of a module |
| `sc_signal` | Observable / reactive variable | Signal wire with change notification |
| `SC_THREAD` | coroutine / Python coroutine (asyncio) | Has its own execution flow, can suspend and resume |
| `SC_METHOD` | event callback | Executes once when triggered by an event, cannot suspend |
| `sc_event` | condition variable / asyncio.Future | Used to notify "something happened" |
| `sc_channel` | typed communication pipe | Communication pipe with protocol |
| `sc_interface` | C++ abstract class / Python ABC | Pure virtual interface defining communication protocol |

Let's expand on each one below.

---

## sc_module -- Reusable Component

`sc_module` is a C++ class that represents a hardware module.

**Software equivalent**: Like a service in a microservice architecture, or a component in React. It has its own internal state, external interfaces, and internal behavioral logic.

```mermaid
classDiagram
    class sc_module {
        <<SystemC base class>>
    }

    class MyModule {
        +sc_in~bool~ clk
        +sc_in~bool~ reset
        +sc_out~int~ result
        -int internal_state
        +SC_CTOR(MyModule)
        -void main_thread()
    }

    sc_module <|-- MyModule
```

**Software world mapping**:

| Hardware Module Element | Software Class Equivalent |
|------------------------|--------------------------|
| input port | constructor parameter / dependency injection |
| output port | return value / callback |
| internal signal | private member variable |
| process (behavior) | method / thread |

---

## sc_port -- Dependency Injection Interface

`sc_port` is a module's external connection point. A port must be connected to a channel that implements a specific interface.

**Software equivalent**: This is **dependency injection**. Instead of creating communication objects directly inside the module, you declare "I need something that implements a certain interface", then inject the concrete implementation from outside.

```mermaid
flowchart LR
    subgraph Producer["Producer Module"]
        P_PORT["sc_port&lt;write_if&gt;<br/>(I need something writable)"]
    end

    subgraph FIFO["fifo Channel"]
        IMPL["Implements write_if"]
    end

    subgraph Consumer["Consumer Module"]
        C_PORT["sc_port&lt;read_if&gt;<br/>(I need something readable)"]
    end

    P_PORT -->|"Bind"| IMPL
    C_PORT -->|"Bind"| IMPL
```

**Key concept**: Modules never communicate directly. They connect to channels via ports, and channels implement specific interfaces. This is like dependency injection (like Python's inject library) with `@inject` -- modules only know the interface, not the concrete implementation.

---

## sc_signal -- Observable Reactive Variable

`sc_signal` is a "wire" that automatically notifies all listeners when its value changes.

**Software equivalent**:

- **RxJS Observable**: When the signal value changes, subscribers are notified
- **Vue.js reactive ref**: You modify the value, the UI updates automatically
- **Database trigger**: A callback fires when a field changes

```mermaid
sequenceDiagram
    participant Writer as Writer
    participant Signal as sc_signal
    participant Reader1 as Listener A
    participant Reader2 as Listener B

    Writer->>Signal: write(42)
    Note over Signal: Value updates at end of delta cycle
    Signal-->>Reader1: value_changed_event triggered
    Signal-->>Reader2: value_changed_event triggered
    Reader1->>Signal: read() == 42
    Reader2->>Signal: read() == 42
```

**Important difference**: sc_signal writes do not take effect immediately. The new value is updated in the next **delta cycle**. This simulates the hardware characteristic of "all signals update simultaneously". (See the delta cycle section in [concurrency-model.md](concurrency-model.md).)

---

## SC_THREAD -- Coroutine / Python Coroutine

`SC_THREAD` is a process with its own execution flow. It can suspend mid-execution (`wait()`) and resume after some event.

**Software equivalent**:

- **Python asyncio coroutine**: Independent lightweight execution flow
- **Python async/await**: `wait()` is `await`
- **C++ coroutine (C++20)**: `co_await` suspends, then resumes later

```mermaid
sequenceDiagram
    participant Kernel as Simulation Kernel
    participant T1 as SC_THREAD (producer)
    participant T2 as SC_THREAD (consumer)

    Kernel->>T1: Start
    T1->>T1: Produce data
    T1->>T1: wait(write_event)
    Note over T1: Suspend, yield control
    Kernel->>T2: Start
    T2->>T2: Read data
    T2->>T2: notify(write_event)
    Note over T2: Suspend
    Kernel->>T1: write_event triggered, resume execution
```

**Key difference**: SC_THREAD uses **cooperative** multitasking, not preemptive. A thread must explicitly call `wait()` to yield control. This means no mutex is needed -- because only one thread is executing at any given moment.

---

## SC_METHOD -- Event Callback

`SC_METHOD` is a simple callback function. When its sensitive event (sensitivity) is triggered, the kernel calls it once. It cannot suspend (cannot call `wait()`).

**Software equivalent**:

- **DOM event listener**: `button.addEventListener('click', handler)`
- **React useEffect**: Executes when dependencies change
- **Database trigger**: Fires on INSERT / UPDATE

```mermaid
flowchart LR
    EVENT["sc_event triggered<br/>(e.g., clock edge)"]
    METHOD["SC_METHOD<br/>handler()"]
    RESULT["Update output"]

    EVENT -->|"Trigger"| METHOD
    METHOD --> RESULT
```

**SC_THREAD vs SC_METHOD**:

| Feature | SC_THREAD | SC_METHOD |
|---------|-----------|-----------|
| Can `wait()` | Yes | No |
| Execution style | Like a coroutine, stateful | Like a callback, stateless |
| Memory overhead | Higher (needs stack) | Lower (no stack needed) |
| Suited for | Complex multi-step workflows | Simple combinational logic, state transitions |
| Software analogy | Python coroutine (asyncio) / async function | event handler / callback |

---

## sc_event -- Condition Variable / asyncio.Future

`sc_event` is the most fundamental synchronization mechanism in SystemC. It represents "something happened".

**Software equivalent**:

- **pthread condition variable**: `pthread_cond_signal` / `pthread_cond_wait`
- **Python asyncio.Future set_result**: When the event fires, waiting threads are woken up
- **Python queue.Queue put**: Unblocks the other end

```mermaid
sequenceDiagram
    participant A as Thread A
    participant E as sc_event
    participant B as Thread B

    B->>E: wait(e)
    Note over B: Suspended...
    A->>E: e.notify()
    E-->>B: Wake up
    B->>B: Continue execution
```

**Three notification timings**:

| Call Style | Takes Effect | Software Analogy |
|-----------|-------------|-----------------|
| `e.notify()` | Immediately (same delta cycle) | `loop.call_soon()` adds to callback queue |
| `e.notify(SC_ZERO_TIME)` | Next delta cycle | `setTimeout(fn, 0)` |
| `e.notify(10, SC_NS)` | After 10 nanoseconds | `setTimeout(fn, 10)` |

---

## sc_channel and sc_interface -- Communication Pipe and Protocol

`sc_interface` defines the communication protocol (pure virtual class), `sc_channel` implements the protocol.

**Software equivalent**:

- `sc_interface` = C++ abstract class / Python ABC
- `sc_channel` = concrete implementation of that interface

```mermaid
classDiagram
    class sc_interface {
        <<abstract>>
    }

    class write_if {
        <<interface>>
        +write(data)* void
    }

    class read_if {
        <<interface>>
        +read(data&)* void
    }

    class fifo_channel {
        -buffer[]
        +write(data) void
        +read(data&) void
    }

    sc_interface <|-- write_if
    sc_interface <|-- read_if
    write_if <|.. fifo_channel
    read_if <|.. fifo_channel
```

**Why this design?**

This follows the software architecture **Interface Segregation Principle**:

- Producer only needs to know `write_if` ("I can write")
- Consumer only needs to know `read_if` ("I can read")
- The actual FIFO implements both, but each end only sees what it needs

This makes modules reusable -- you can replace the FIFO with any channel that implements `write_if`, and the Producer needs no modification.

---

## Simulation Kernel -- Event Loop

SystemC's simulation kernel is an event loop, almost identical in concept to the Python asyncio event loop.

```mermaid
flowchart TD
    INIT["Initialization<br/>Create modules, bind ports"]
    EVAL["Evaluate Phase<br/>Execute all triggered processes"]
    UPDATE["Update Phase<br/>Update all signal values"]
    DELTA{"More pending<br/>delta cycle events?"}
    TIME{"More future<br/>timed events?"}
    ADVANCE["Advance simulation time<br/>to next event"]
    END["Simulation ends"]

    INIT --> EVAL
    EVAL --> UPDATE
    UPDATE --> DELTA
    DELTA -->|"Yes"| EVAL
    DELTA -->|"No"| TIME
    TIME -->|"Yes"| ADVANCE
    ADVANCE --> EVAL
    TIME -->|"No"| END
```

**Comparison with Python asyncio event loop**:

| Concept | Python asyncio | SystemC |
|---------|---------|---------|
| Event loop | Event Loop | Simulation Kernel |
| Callback queue | Callback Queue | Pending SC_METHODs |
| Microtask | call_soon queue | Delta Cycle |
| Timer | call_later | timed event (wait 10 ns) |
| I/O callback | add_reader callback | SC_THREAD woken by event |

---

## Delta Cycle -- Microtask Queue

Delta cycle is one of the most confusing concepts for software engineers in SystemC. Its core purpose is to **resolve ordering issues with simultaneous updates**.

### The Problem: Signal Read/Write Ordering

Suppose two processes execute at the same time:
- Process A reads signal X, then writes to signal Y based on the result
- Process B reads signal Y, then writes to signal X based on the result

Without delta cycles, the result would depend on who executes first, A or B (race condition).

### The Solution: Separate "Compute" and "Update"

```mermaid
sequenceDiagram
    participant K as Kernel
    participant A as Process A
    participant B as Process B
    participant S as Signals

    rect rgb(230, 240, 255)
        Note over K,S: Delta Cycle 0 - Evaluate
        K->>A: Execute
        A->>S: Read old values, compute new values
        A->>S: Write new values (queued, not yet effective)
        K->>B: Execute
        B->>S: Read old values, compute new values
        B->>S: Write new values (queued)
    end

    rect rgb(255, 240, 230)
        Note over K,S: Delta Cycle 0 - Update
        K->>S: Batch update all signals to new values
    end

    rect rgb(230, 255, 230)
        Note over K,S: Delta Cycle 1 - Evaluate
        Note over K: If signal values changed,<br/>trigger sensitive processes to run again
    end
```

**Software analogy**:

- Like React's `setState` -- when you call `setState`, the value doesn't change immediately; it updates together after the render cycle ends
- Or like Vue.js's `nextTick` -- DOM updates are batched

**Why does this matter?**

Because in hardware, all registers update "simultaneously" at the clock edge. The delta cycle mechanism lets the simulator correctly simulate this "simultaneous" semantics on a single thread.

(For more detailed explanation, see [concurrency-model.md](concurrency-model.md).)

---

## Lifecycle of a Complete Example

The following is the complete lifecycle of a typical SystemC model from start to finish:

```mermaid
sequenceDiagram
    participant Main as sc_main()
    participant Kernel as Simulation Kernel
    participant Mod as Modules
    participant Proc as Processes

    Main->>Mod: 1. Create module instances
    Main->>Mod: 2. Connect ports to channels
    Main->>Kernel: 3. sc_start() launches simulation
    Kernel->>Mod: 4. Call before_end_of_elaboration()
    Kernel->>Mod: 5. Call end_of_elaboration()
    Kernel->>Proc: 6. Start all SC_THREADs (execute until first wait)
    Kernel->>Proc: 7. Execute all triggered SC_METHODs

    loop Simulation loop
        Kernel->>Proc: Evaluate: execute ready processes
        Kernel->>Kernel: Update: update signals
        Kernel->>Kernel: Delta cycle convergence
        Kernel->>Kernel: Advance simulation time
    end

    Kernel->>Mod: end_of_simulation()
    Kernel->>Main: sc_start() returns
```

---

## Concept Panorama

```mermaid
flowchart TB
    subgraph Module_A["sc_module: Producer"]
        T1["SC_THREAD<br/>main_thread()"]
        P1["sc_port&lt;write_if&gt;"]
    end

    subgraph Channel["sc_channel: FIFO"]
        BUF["buffer[]"]
        WE["write_event"]
        RE["read_event"]
    end

    subgraph Module_B["sc_module: Consumer"]
        T2["SC_THREAD<br/>main_thread()"]
        P2["sc_port&lt;read_if&gt;"]
    end

    subgraph Kernel["Simulation Kernel (Event Loop)"]
        EL["Event Loop"]
        DC["Delta Cycle Manager"]
        TC["Time Controller"]
    end

    T1 --> P1
    P1 -->|"write()"| Channel
    Channel -->|"read()"| P2
    P2 --> T2

    WE -->|"Notify"| Kernel
    RE -->|"Notify"| Kernel
    Kernel -->|"Wake up"| T1
    Kernel -->|"Wake up"| T2
```

---

## Next Steps

- Want to start with hands-on examples? Go to [learning-path.md](learning-path.md) to choose your learning track
- Want to dive deeper into the concurrency model? Read [concurrency-model.md](concurrency-model.md)
- Want to learn about TLM transaction level model? Read [tlm-explained.md](tlm-explained.md)
- Want to understand Behavioral vs RTL? Read [behavioral-vs-rtl.md](behavioral-vs-rtl.md)
