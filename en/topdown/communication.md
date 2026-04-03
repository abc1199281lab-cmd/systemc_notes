# Communication Mechanisms

## Everyday Analogy: Inter-Department Communication in a Company

Imagine how different departments communicate within a large company:

- **Interface** = The company's communication rules — "What format should reports use? How do you book a meeting?"
- **Channel** = The actual communication tools — Email, Slack, phone
- **Port** = Each department's external contact window — "To reach Sales, dial extension 101"
- **Signal** = A bulletin board — post a new notice, and everyone sees it the next time they walk by
- **FIFO** = A queued mailbox — letters that arrive first get processed first
- **Mutex** = The meeting room key — only one person can use it at a time
- **Semaphore** = The number of remaining parking spaces — a limited quantity of shared resources

The key point is: departments don't need to know what computers or software the other side uses.
They only need to follow the agreed-upon communication rules (the interface).

---

## The Interface-Channel-Port Pattern

This is the core design pattern of SystemC's communication mechanism,
and the key to understanding all communication components.

```mermaid
flowchart TD
    subgraph "Design Principles"
        IF["Interface<br/>Defines 'what can be done'<br/>(pure virtual class)"]
        CH["Channel<br/>Implements 'how to do it'<br/>(inherits Interface)"]
        PO["Port<br/>Declares 'what I need'<br/>(template parameter is Interface)"]
    end

    PO -->|"Calls through Interface"| CH
    CH -->|"Implements"| IF

    style IF fill:#e3f2fd
    style CH fill:#fff3e0
    style PO fill:#c8e6c9
```

### Why This Design?

```mermaid
flowchart LR
    subgraph "Bad Design ✗"
        A1["Module A"] -->|"Direct access"| B1["Module B internals"]
    end

    subgraph "SystemC Design ✓"
        A2["Module A<br/>Port"] -->|"Through Interface"| C2["Channel"]
        C2 -->|"Through Interface"| B2["Module B<br/>Port"]
    end

    style A1 fill:#ffcdd2
    style B1 fill:#ffcdd2
    style A2 fill:#c8e6c9
    style C2 fill:#e3f2fd
    style B2 fill:#c8e6c9
```

Benefits:
1. **Decoupling** — Modules don't need to know about each other's existence
2. **Swappable** — Replace a Channel implementation without modifying the modules
3. **Reusable** — The same module can be used in different systems

---

## Signal and the Request-Update Mechanism

`sc_signal` is the most basic communication channel, corresponding to a wire in hardware.

### Core Behavior of Signal

```mermaid
sequenceDiagram
    participant P as Process
    participant S as sc_signal
    participant E as Simulation Engine

    Note over S: Current value = 0, New value = 0

    P->>S: write(1)
    Note over S: Current value = 0, New value = 1<br/>(New value buffered, not yet effective!)
    S->>E: request_update()

    Note over E: Evaluate phase ends
    Note over E: Enter Update phase

    E->>S: update()
    Note over S: Current value = 1, New value = 1<br/>(Now it takes effect!)
    S->>E: Value changed! Notify related processes
```

### Why Can't Signal Update Immediately?

```mermaid
graph TD
    subgraph "If updated immediately (Wrong!)"
        A1["Process A writes sig = 1"]
        B1["Process B reads sig = 1"]
        C1["Process C reads sig = 1"]
        A1 --> B1 --> C1
        NOTE1["The value B and C see depends on<br/>execution order! Non-deterministic!"]
    end

    subgraph "Request-Update (Correct)"
        A2["Process A writes sig (buffered)"]
        B2["Process B reads sig = old value"]
        C2["Process C reads sig = old value"]
        U2["Update: sig updated to new value"]
        A2 --> U2
        B2 --> U2
        C2 --> U2
        NOTE2["B and C both see the old value.<br/>Regardless of execution order, result is consistent!"]
    end

    style NOTE1 fill:#ffcdd2
    style NOTE2 fill:#c8e6c9
```

### Class Structure of sc_signal

```mermaid
classDiagram
    class sc_interface {
        <<abstract>>
        +register_port()
        +default_event() sc_event
    }

    class sc_signal_in_if~T~ {
        <<abstract>>
        +read() T
        +value_changed_event() sc_event
    }

    class sc_signal_inout_if~T~ {
        <<abstract>>
        +write(T)
    }

    class sc_prim_channel {
        #request_update()
        #update()
        #async_request_update()
    }

    class sc_signal~T~ {
        -m_cur_val : T
        -m_new_val : T
        -m_change_event : sc_event
        +read() T
        +write(T)
        #update()
    }

    sc_interface <|-- sc_signal_in_if
    sc_signal_in_if <|-- sc_signal_inout_if
    sc_prim_channel <|-- sc_signal
    sc_signal_inout_if <|.. sc_signal
```

---

## FIFO — First-In First-Out Queue

`sc_fifo` is like queuing to buy something — whoever lines up first gets served first.

```mermaid
flowchart LR
    W["Write end<br/>(Producer)"] -->|"write(data)"| FIFO["sc_fifo<br/>Capacity = N<br/>[data3][data2][data1]"]
    FIFO -->|"read()"| R["Read end<br/>(Consumer)"]

    FULL["Full? → write blocks<br/>(waits until space is available)"]
    EMPTY["Empty? → read blocks<br/>(waits until data is available)"]

    FIFO -.-> FULL
    FIFO -.-> EMPTY

    style W fill:#c8e6c9
    style R fill:#fff3e0
    style FIFO fill:#e3f2fd
```

### FIFO Event Notifications

```mermaid
sequenceDiagram
    participant Producer
    participant FIFO as sc_fifo (Capacity=2)
    participant Consumer

    Producer->>FIFO: write("A")
    Note over FIFO: ["A"]
    FIFO->>Consumer: data_written_event notification

    Producer->>FIFO: write("B")
    Note over FIFO: ["A", "B"] (Full!)

    Producer->>FIFO: write("C") — Blocks!
    Note over Producer: Waiting for space...

    Consumer->>FIFO: read() → "A"
    Note over FIFO: ["B"]
    FIFO->>Producer: data_read_event notification
    Note over Producer: Space available!

    Producer->>FIFO: write("C") — Success
    Note over FIFO: ["B", "C"]
```

### Differences Between FIFO and Signal

| Property | sc_signal | sc_fifo |
|----------|-----------|---------|
| Data retention | Only keeps the latest value | Keeps all values until read |
| Blocking | Non-blocking | Blocks on write when full, blocks on read when empty |
| Use case | Hardware signal wire | Data streaming, producer-consumer |
| Multiple readers | Yes | No (reading consumes data) |
| Delta cycle | Required (request-update) | Required |

---

## Mutex — Mutual Exclusion Lock

`sc_mutex` ensures only one process can access a shared resource at the same time.

```mermaid
sequenceDiagram
    participant A as Process A
    participant M as sc_mutex
    participant B as Process B

    A->>M: lock() — Success!
    Note over M: Locked (owner: A)

    B->>M: lock() — Blocks!
    Note over B: Waiting...

    A->>M: unlock()
    Note over M: Unlocked
    M->>B: Your turn!

    B->>M: lock() — Success!
    Note over M: Locked (owner: B)
```

Analogy: A library study room has only one key.
When you're done, you return the key, and the next person in line can enter.

---

## Semaphore

`sc_semaphore` allows up to N processes to access a resource simultaneously.

```mermaid
flowchart TD
    SEM["sc_semaphore<br/>Initial value = 3"]
    P1["Process 1<br/>wait() → 2 remaining"]
    P2["Process 2<br/>wait() → 1 remaining"]
    P3["Process 3<br/>wait() → 0 remaining"]
    P4["Process 4<br/>wait() → Blocks!"]

    SEM --> P1
    SEM --> P2
    SEM --> P3
    SEM -.->|"No slots left"| P4

    P1 -->|"post() → 1 remaining"| P4_OK["Process 4 can proceed"]

    style SEM fill:#e3f2fd
    style P4 fill:#ffcdd2
    style P4_OK fill:#c8e6c9
```

Analogy: A parking lot has 3 spaces. The first three cars can park directly.
The fourth car must wait at the entrance until a car leaves and frees up a space.

---

## Resolved Signal — Multi-Driver Signal

When multiple processes drive the same signal wire simultaneously, a "resolution" rule is needed:

```mermaid
flowchart TD
    D1["Driver 1<br/>Output: 1"] --> RS["sc_signal_resolved<br/>Resolution Logic"]
    D2["Driver 2<br/>Output: Z (High Impedance)"] --> RS
    D3["Driver 3<br/>Output: 1"] --> RS

    RS --> RESULT["Resolved Result: 1"]

    style RS fill:#e3f2fd
    style RESULT fill:#c8e6c9
```

### Four-Value Logic Resolution Table

| Driver A | Driver B | Resolved Result |
|----------|----------|-----------------|
| 0 | 0 | 0 |
| 0 | 1 | X (Conflict!) |
| 0 | Z | 0 |
| 1 | 1 | 1 |
| 1 | Z | 1 |
| Z | Z | Z |

A regular `sc_signal` allows only one writer (writer policy).
`sc_signal_resolved` allows multiple writers but performs logic resolution.

---

## How Communication Components Map to Hardware

```mermaid
graph LR
    subgraph "SystemC Communication Components"
        SIG["sc_signal"]
        FIFO_SC["sc_fifo"]
        MUX_SC["sc_mutex"]
        SEM_SC["sc_semaphore"]
        RES["sc_signal_resolved"]
    end

    subgraph "Hardware Equivalents"
        WIRE["Wire"]
        BUFFER["Buffer / FIFO"]
        ARB["Arbiter"]
        RESOURCE["Shared Resource Controller"]
        BUS["Tri-state Bus"]
    end

    SIG -->|"Maps to"| WIRE
    FIFO_SC -->|"Maps to"| BUFFER
    MUX_SC -->|"Maps to"| ARB
    SEM_SC -->|"Maps to"| RESOURCE
    RES -->|"Maps to"| BUS

    style SIG fill:#e3f2fd
    style FIFO_SC fill:#e3f2fd
    style MUX_SC fill:#e3f2fd
    style SEM_SC fill:#e3f2fd
    style RES fill:#e3f2fd
    style WIRE fill:#fff3e0
    style BUFFER fill:#fff3e0
    style ARB fill:#fff3e0
    style RESOURCE fill:#fff3e0
    style BUS fill:#fff3e0
```

---

## Complete Communication Architecture Diagram

```mermaid
classDiagram
    class sc_interface {
        <<abstract>>
    }

    class sc_signal_in_if~T~ {
        <<abstract>>
        +read() T
    }

    class sc_signal_inout_if~T~ {
        <<abstract>>
        +write(T)
    }

    class sc_fifo_in_if~T~ {
        <<abstract>>
        +read(T)
        +nb_read(T) bool
        +num_available() int
    }

    class sc_fifo_out_if~T~ {
        <<abstract>>
        +write(T)
        +nb_write(T) bool
        +num_free() int
    }

    class sc_mutex_if {
        <<abstract>>
        +lock() int
        +trylock() int
        +unlock() int
    }

    class sc_semaphore_if {
        <<abstract>>
        +wait() int
        +trywait() int
        +post() int
        +get_value() int
    }

    sc_interface <|-- sc_signal_in_if
    sc_signal_in_if <|-- sc_signal_inout_if
    sc_interface <|-- sc_fifo_in_if
    sc_interface <|-- sc_fifo_out_if
    sc_interface <|-- sc_mutex_if
    sc_interface <|-- sc_semaphore_if

    class sc_signal~T~ {
        +read() T
        +write(T)
    }
    class sc_fifo~T~ {
        +read(T)
        +write(T)
    }
    class sc_mutex {
        +lock()
        +unlock()
    }
    class sc_semaphore {
        +wait()
        +post()
    }

    sc_signal_inout_if <|.. sc_signal
    sc_fifo_in_if <|.. sc_fifo
    sc_fifo_out_if <|.. sc_fifo
    sc_mutex_if <|.. sc_mutex
    sc_semaphore_if <|.. sc_semaphore
```

---

## Related Modules

| Concept | File | Relationship |
|---------|------|--------------|
| Module Hierarchy | [hierarchy.md](hierarchy.md) | Port and Export are the external interfaces of modules |
| Event Mechanism | [events.md](events.md) | Channels notify value changes through events |
| Scheduling Mechanism | [scheduling.md](scheduling.md) | request_update/update is a core part of scheduling |
| Data Types | [datatypes.md](datatypes.md) | The template parameter of Signal determines what data is transmitted |
| TLM | [tlm.md](tlm.md) | TLM is a higher-level communication abstraction |

### Corresponding Source Code Files

| Source Code Concept | Code File |
|---------------------|-----------|
| sc_interface | [doc_v2/code/sysc/communication/sc_interface.md](../code/sysc/communication/sc_interface.md) |
| sc_signal | [doc_v2/code/sysc/communication/sc_signal.md](../code/sysc/communication/sc_signal.md) |
| sc_signal_ifs | [doc_v2/code/sysc/communication/sc_signal_ifs.md](../code/sysc/communication/sc_signal_ifs.md) |
| sc_prim_channel | [doc_v2/code/sysc/communication/sc_prim_channel.md](../code/sysc/communication/sc_prim_channel.md) |
| sc_port | [doc_v2/code/sysc/communication/sc_port.md](../code/sysc/communication/sc_port.md) |
| sc_export | [doc_v2/code/sysc/communication/sc_export.md](../code/sysc/communication/sc_export.md) |
| sc_fifo | [doc_v2/code/sysc/communication/sc_fifo.md](../code/sysc/communication/sc_fifo.md) |
| sc_mutex | [doc_v2/code/sysc/communication/sc_mutex.md](../code/sysc/communication/sc_mutex.md) |
| sc_semaphore | [doc_v2/code/sysc/communication/sc_semaphore.md](../code/sysc/communication/sc_semaphore.md) |
| sc_signal_resolved | [doc_v2/code/sysc/communication/sc_signal_resolved.md](../code/sysc/communication/sc_signal_resolved.md) |

---

## Learning Tips

1. **Interface-Channel-Port is the most important design pattern in SystemC** — understand it, and you've understood half of SystemC
2. **Signal values don't update immediately** — after a write, the new value only takes effect in the next delta cycle. This is the most common source of confusion for beginners
3. **FIFO blocks, Signal does not** — choosing the wrong communication component can cause unexpected behavior in your design
4. **A regular signal allows only one writer** — if you need multiple drivers, use `sc_signal_resolved`
5. **Mutex and Semaphore are mainly used in abstract modeling** — RTL-level designs typically don't use them
6. **The Port's `->` operator forwards to the Channel's Interface** — `port->read()` actually calls the Channel's `read()`
