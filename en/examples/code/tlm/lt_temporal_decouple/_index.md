# LT + Temporal Decoupling Example Overview

## Software Analogy: Batch Processing and Time Budgets

Imagine a multiplayer online game server. Ideally, every player's action should be synchronized to all other players "in real time," but synchronizing for every small action is too expensive in terms of performance.

The practical approach is: each player accumulates actions over a short period (e.g., 100ms), then synchronizes them all at once. This "allowed out-of-sync time" is called the **time quantum**. As long as it does not exceed this time, even if the order of players' actions is slightly off, players will not notice.

TLM's Temporal Decoupling is the same concept:

| Game Server | TLM Temporal Decoupling |
|---|---|
| Accumulate actions, synchronize periodically | Initiator accumulates local time, synchronizes with global time periodically |
| Time quantum = 100ms | Time quantum = configured simulation time interval |
| Reduces synchronization frequency, improves performance | Reduces `sc_core::wait()` calls, improves simulation speed |
| Synchronization point = tick | Synchronization point = quantum keeper triggers `wait()` |

## Why Is Temporal Decoupling Needed?

In standard LT mode, after each `b_transport()` completes, the initiator calls `wait(delay)` to consume simulation time. Each `wait()` causes the SystemC kernel to perform a context switch, which is the main bottleneck for simulation performance.

The temporal decoupling approach is:

1. Instead of calling `wait()` immediately, accumulate the delay into a **local time offset**
2. When the accumulated local time exceeds the configured **time quantum**, perform a single `wait()` to synchronize back to global time
3. This significantly reduces the number of context switches

## System Architecture

```mermaid
graph LR
    subgraph TD_Initiator["td_initiator_top (ID=101)"]
        TG1["traffic_generator"]
        LTDI["lt_td_initiator<br/>(with quantum keeper)"]
        TG1 -->|request_fifo| LTDI
        LTDI -->|response_fifo| TG1
    end

    subgraph Normal_Initiator["initiator_top (ID=102)"]
        TG2["traffic_generator"]
        LI["lt_initiator<br/>(no temporal decoupling)"]
        TG2 -->|request_fifo| LI
        LI -->|response_fifo| TG2
    end

    subgraph Bus["SimpleBusLT<2,2>"]
        Router["Address Router"]
    end

    subgraph Target_1["lt_synch_target (ID=201)"]
        MEM1["Memory 4KB<br/>(forced synchronization)"]
    end

    subgraph Target_2["lt_target (ID=202)"]
        MEM2["Memory 4KB<br/>(regular LT)"]
    end

    LTDI -->|initiator_socket| Router
    LI -->|initiator_socket| Router
    Router --> MEM1
    Router --> MEM2
```

Note: this example intentionally mixes a temporal decoupling initiator with a regular LT initiator, and a target that forces synchronization (`lt_synch_target`) with a regular target, to demonstrate how different components can coexist.

## Quantum Keeper Operating Principle

```mermaid
sequenceDiagram
    participant Init as lt_td_initiator
    participant QK as Quantum Keeper
    participant Kernel as SystemC Kernel

    Note over Init,Kernel: Quantum = 200 ns

    Init->>Init: b_transport() returns delay=50ns
    Init->>QK: set(local_time + 50ns)
    QK->>QK: local_time = 50ns<br/>not yet exceeding quantum

    Init->>Init: b_transport() returns delay=80ns
    Init->>QK: set(local_time + 80ns)
    QK->>QK: local_time = 130ns<br/>not yet exceeding quantum

    Init->>Init: b_transport() returns delay=90ns
    Init->>QK: set(local_time + 90ns)
    QK->>QK: local_time = 220ns<br/>exceeds quantum!
    QK->>Kernel: wait(220ns) -- synchronize global time
    QK->>QK: local_time = 0ns
```

## Source Files

| File | Description |
|---|---|
| `src/lt_temporal_decouple.cpp` | Program entry point `sc_main` |
| `include/lt_temporal_decouple_top.h` / `src/lt_temporal_decouple_top.cpp` | Top-level module |
| `include/td_initiator_top.h` / `src/td_initiator_top.cpp` | Temporal decoupling initiator wrapper module |
| `include/initiator_top.h` / `src/initiator_top.cpp` | Regular LT initiator wrapper module (for comparison) |

For detailed source code analysis, see [lt-temporal-decouple.md](lt-temporal-decouple.md).
