# Scheduling

## Everyday Analogy: A Restaurant Kitchen's Order System

Imagine the kitchen of a busy restaurant:

- **Orders** = event notifications — "Table 3 wants a steak!"
- **Service window** = Runnable Queue — all dishes ready to go are lined up here
- **Chef serves dishes one by one** = Evaluate Phase — completing them one at a time
- **Waiter double-checks the dishes** = Update Phase — making sure everything is correct before serving
- **One round of serving completed** = one Delta Cycle
- **Waiting for the next batch of orders** = advancing simulation time

All orders in the same round arrived "at the same time" (same delta),
but the chef can only prepare them one at a time (single-threaded simulation),
so SystemC needs a fair set of scheduling rules.

---

## The Evaluate-Update Paradigm

The core of SystemC scheduling is the continuous repetition of two phases:

```mermaid
flowchart LR
    E["Evaluate Phase<br/>Execute processes"] --> U["Update Phase<br/>Update channels"]
    U --> E2["Evaluate Phase<br/>Execute awakened processes"]
    E2 --> U2["Update Phase<br/>Update again"]
    U2 --> NEXT["...Repeat until stable..."]

    style E fill:#e3f2fd
    style U fill:#fff3e0
    style E2 fill:#e3f2fd
    style U2 fill:#fff3e0
```

### Why Two Separate Phases?

Because of a fundamental hardware property: **all flip-flops toggle simultaneously at the clock edge**.

If one process were allowed to write a value and another process could immediately see the new value,
then execution order would affect results — this does not match hardware behavior.

Separating into Evaluate and Update guarantees:
1. During the Evaluate phase, all processes read **the results from the previous Update**
2. Newly written values are "buffered" temporarily
3. Only after all processes have finished running are the values updated uniformly in the Update phase

---

## Delta Cycle: A Complete Breakdown

A delta cycle consists of one Evaluate + one Update:

```mermaid
flowchart TD
    subgraph "Delta Cycle #0"
        E0[/"Evaluate Phase<br/>Take processes from runnable queue<br/>Execute in order"/]
        U0[/"Update Phase<br/>All channels with request_update<br/>Apply new values"/]
        E0 --> U0
    end

    subgraph "Delta Cycle #1"
        E1[/"Evaluate Phase<br/>Processes awakened by delta notifications"/]
        U1[/"Update Phase<br/>Apply updates"/]
        E1 --> U1
    end

    U0 -->|"Channel value changed<br/>Triggers delta notification"| E1

    subgraph "Time Advancement"
        ADV[/"Advance to next<br/>timed event"/]
    end

    U1 -->|"No more delta notifications"| ADV

    style E0 fill:#e3f2fd
    style U0 fill:#fff3e0
    style E1 fill:#e3f2fd
    style U1 fill:#fff3e0
    style ADV fill:#e8f5e9
```

### A Concrete Example

Suppose there is a simple combinational logic chain: A → B → C

```
signal_a changes → process_b reads a, computes b → signal_b changes → process_c reads b, computes c
```

```mermaid
sequenceDiagram
    participant SA as signal_a
    participant PB as process_b
    participant SB as signal_b
    participant PC as process_c
    participant SC2 as signal_c

    Note over SA, SC2: Time = 10ns

    Note over SA, SC2: Delta #0
    Note over SA: signal_a set to 1 externally
    SA->>PB: Value change notification
    PB->>PB: Read signal_a = 1, compute
    PB->>SB: Write signal_b (request_update)

    Note over SA, SC2: Update
    SB->>SB: Value updated from 0 to 1

    Note over SA, SC2: Delta #1
    SB->>PC: Value change notification
    PC->>PC: Read signal_b = 1, compute
    PC->>SC2: Write signal_c (request_update)

    Note over SA, SC2: Update
    SC2->>SC2: Value updated

    Note over SA, SC2: Delta #2 (No more notifications, delta cycle ends)
    Note over SA, SC2: Advance time to next event
```

---

## Process Execution Order

### Key Concept: Within the same delta, the execution order of processes is **nondeterministic**

```mermaid
graph TD
    Q[Runnable Queue<br/>Contains Process A, B, C] --> NOTE["The engine may execute<br/>A, B, C in any order"]
    NOTE --> O1["Order 1: A → B → C"]
    NOTE --> O2["Order 2: B → A → C"]
    NOTE --> O3["Order 3: C → B → A"]
    NOTE --> OX["... other permutations ..."]

    O1 --> RES["But thanks to Evaluate-Update separation<br/>regardless of order<br/>the result is the same!"]
    O2 --> RES
    O3 --> RES
    OX --> RES

    style RES fill:#c8e6c9
    style Q fill:#e3f2fd
```

**This is the elegance of the Evaluate-Update design**:
Because all processes read old values during the Evaluate phase,
and newly written values only take effect during Update,
no matter which process the engine executes first, the final result is the same.

### Scheduling Differences Between SC_METHOD and SC_THREAD

```mermaid
flowchart TD
    subgraph "SC_METHOD Lifecycle"
        M1[Awakened by event] --> M2[Execute from start to finish]
        M2 --> M3[return]
        M3 --> M4[Wait for next event]
        M4 --> M1
    end

    subgraph "SC_THREAD Lifecycle"
        T1[First launch] --> T2[Execute until wait]
        T2 --> T3[Suspend, save state]
        T3 --> T4[Awakened by event]
        T4 --> T5[Resume after wait]
        T5 --> T2
    end

    style M1 fill:#e3f2fd
    style T1 fill:#fff3e0
```

---

## How the Runnable Queue Works

The Runnable Queue is the core data structure of the scheduler,
storing all processes that are "ready to execute":

```mermaid
classDiagram
    class sc_runnable {
        -m_methods_push_head : sc_method_process*
        -m_methods_pop : sc_method_process*
        -m_threads_push_head : sc_thread_process*
        -m_threads_pop : sc_thread_process*
        +init()
        +is_empty() bool
        +is_initialized() bool
        +push_back_method(method)
        +push_back_thread(thread)
        +pop_method() sc_method_process*
        +pop_thread() sc_thread_process*
        +remove_method(method)
        +remove_thread(thread)
    }

    note for sc_runnable "Uses two pointers to separate<br/>push and pop operations<br/>Prevents newly added processes<br/>from affecting the current round<br/>during the evaluate phase"
```

### Push / Pop Separation Mechanism

```mermaid
flowchart TD
    subgraph "Evaluate Phase"
        POP["Take processes from Pop end<br/>Execute in order"]
        PUSH["Newly awakened processes<br/>Added to Push end"]
    end

    subgraph "Between Rounds"
        SWAP["After Pop end is empty<br/>Push end becomes next round's Pop end"]
    end

    POP --> SWAP
    PUSH --> SWAP
    SWAP --> POP

    style POP fill:#e3f2fd
    style PUSH fill:#fff3e0
    style SWAP fill:#e8f5e9
```

This design ensures that processes awakened by immediate notifications during the evaluate phase
will be executed within the **same delta cycle**, without disrupting the currently executing order.

---

## Scheduling of Timed vs Untimed Notifications

```mermaid
flowchart TD
    EVENT["Event Notification"] --> TYPE{Notification type?}

    TYPE -->|"notify()"| IMM["Immediate notification<br/>Added to current delta's<br/>runnable queue"]
    TYPE -->|"notify(SC_ZERO_TIME)"| DELTA["Delta notification<br/>Added to next delta's<br/>runnable queue"]
    TYPE -->|"notify(t)"| TIMED["Timed notification<br/>Added to timed event queue<br/>Sorted by time"]

    IMM --> EVAL_NOW["Executed in current<br/>evaluate phase"]
    DELTA --> EVAL_NEXT["Executed in next<br/>evaluate phase"]
    TIMED --> TIME_ADV["Waits for time<br/>advancement to execute"]

    style IMM fill:#ffcdd2
    style DELTA fill:#fff3e0
    style TIMED fill:#e3f2fd
```

### Timed Event Queue

Timed events are managed using a priority queue, sorted by trigger time:

```mermaid
graph TD
    subgraph "Timed Event Queue (Priority Queue)"
        T1["t=10ns: Clock rising edge"]
        T2["t=15ns: Data arrival"]
        T3["t=20ns: Clock falling edge"]
        T4["t=20ns: Timeout timer"]
    end

    T1 --> T2 --> T3 --> T4

    CURRENT["Current simulation time t=5ns"]
    CURRENT -.->|"Advance to"| T1

    style CURRENT fill:#c8e6c9
    style T1 fill:#ffcdd2
```

When all delta cycles have stabilized (no more delta notifications),
the engine advances simulation time to the next time point in the timed event queue.

---

## The Complete Scheduling Algorithm

```mermaid
flowchart TD
    START([Start]) --> INIT["Initialize<br/>Add all processes to runnable queue"]
    INIT --> EVAL

    EVAL["EVALUATE<br/>Take one process from runnable queue and execute"]
    EVAL --> MORE_P{Runnable queue<br/>still has processes?}
    MORE_P -->|Yes| EVAL

    MORE_P -->|No| UPDATE["UPDATE<br/>Update all channels with request_update"]
    UPDATE --> DELTA_N{Any delta<br/>notifications?}

    DELTA_N -->|Yes| ADD_DELTA["Add notified processes<br/>to runnable queue"]
    ADD_DELTA --> EVAL

    DELTA_N -->|No| TNOTIFY{"Any timed<br/>notifications?<br/>And not past<br/>specified time?"}

    TNOTIFY -->|Yes| ADVANCE["Advance simulation time<br/>to nearest timed event"]
    ADVANCE --> FIRE["Fire all events at that time<br/>Add related processes to runnable queue"]
    FIRE --> EVAL

    TNOTIFY -->|No| END([Simulation ends])

    style START fill:#c8e6c9
    style END fill:#ffcdd2
    style EVAL fill:#e3f2fd
    style UPDATE fill:#fff3e0
    style ADVANCE fill:#e8f5e9
```

---

## Common Scheduling Pitfalls

### Pitfall 1: Infinite Delta Cycle

```cpp
// Dangerous! A and B trigger each other, never stopping
SC_METHOD(process_a);
sensitive << sig_b;
void process_a() { sig_a.write(!sig_b.read()); }

SC_METHOD(process_b);
sensitive << sig_a;
void process_b() { sig_b.write(!sig_a.read()); }
```

```mermaid
flowchart LR
    A["Process A<br/>sig_a = !sig_b"] -->|"sig_a changes"| B["Process B<br/>sig_b = !sig_a"]
    B -->|"sig_b changes"| A

    style A fill:#ffcdd2
    style B fill:#ffcdd2
```

The SystemC engine will report an error and abort when it detects too many delta cycles.

### Pitfall 2: Calling wait() Inside SC_METHOD

```cpp
SC_METHOD(my_method);
void my_method() {
    wait(10, SC_NS);  // Error! SC_METHOD cannot call wait!
}
```

SC_METHOD does not have its own execution stack, so it cannot suspend and resume.
Only SC_THREAD and SC_CTHREAD can call `wait()`.

---

## Related Modules

| Concept | File | Relationship |
|---------|------|-------------|
| Simulation Engine | [simulation-engine.md](simulation-engine.md) | The scheduler is the heart of the simulation engine |
| Event Mechanism | [events.md](events.md) | Events trigger scheduling |
| Communication Mechanism | [communication.md](communication.md) | Channel's request_update/update is part of scheduling |
| Module Hierarchy | [hierarchy.md](hierarchy.md) | Processes are defined within modules |

### Corresponding Source Code Files

| Source Code Concept | Code File |
|---------------------|-----------|
| sc_simcontext | [doc_v2/code/sysc/kernel/sc_simcontext.md](../code/sysc/kernel/sc_simcontext.md) |
| sc_runnable | [doc_v2/code/sysc/kernel/sc_runnable.md](../code/sysc/kernel/sc_runnable.md) |
| sc_method_process | [doc_v2/code/sysc/kernel/sc_method_process.md](../code/sysc/kernel/sc_method_process.md) |
| sc_thread_process | [doc_v2/code/sysc/kernel/sc_thread_process.md](../code/sysc/kernel/sc_thread_process.md) |
| sc_event | [doc_v2/code/sysc/kernel/sc_event.md](../code/sysc/kernel/sc_event.md) |
| sc_prim_channel | [doc_v2/code/sysc/communication/sc_prim_channel.md](../code/sysc/communication/sc_prim_channel.md) |

---

## Learning Tips

1. **Evaluate-Update is the soul of scheduling** — understand this and you understand why SystemC can correctly simulate hardware
2. **Execution order within the same delta is nondeterministic** — never rely on process execution order to write correct code
3. **Delta cycles are fast but not free** — too many delta cycles (e.g., overly long combinational logic chains) will slow down simulation
4. **Time only advances after delta cycles stabilize** — simulation time moves in "discrete jumps," not as a continuous flow
5. **Beginners should master SC_THREAD + wait() first** — it is more intuitive than SC_METHOD + static sensitivity
6. **Draw timing diagrams for debugging** — when facing scheduling issues, manually sketch out the values each process sees in every delta cycle
