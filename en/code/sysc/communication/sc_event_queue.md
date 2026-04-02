# sc_event_queue -- Event Queue

## Overview

`sc_event_queue` is a queue that can hold multiple pending notifications simultaneously. Unlike a regular `sc_event` (where a later `notify()` overwrites the previous one), `sc_event_queue` guarantees that **each `notify()` call produces a corresponding trigger**.

**Source files:** `sc_event_queue.h`, `sc_event_queue.cpp`

## Everyday Analogy

Comparing `sc_event` and `sc_event_queue`:

- **sc_event** is like "one alarm clock" -- you set three alarms in a row, but each setting overwrites the previous one, so it only rings once
- **sc_event_queue** is like an "alarm schedule" -- you set three alarm times, and all three will ring at their respective times

If multiple notifications are scheduled for the same time point, the event queue will fire them in separate delta cycles, ensuring each notification is seen by the process.

## Class Structure

```mermaid
classDiagram
    sc_interface <|.. sc_event_queue_if
    sc_event_queue_if <|.. sc_event_queue
    sc_module <|-- sc_event_queue

    class sc_event_queue_if {
        <<interface>>
        +notify(double, sc_time_unit)*
        +notify(const sc_time&)*
        +cancel_all()*
    }

    class sc_event_queue {
        -sc_ppq~sc_time*~ m_ppq
        -sc_event m_e
        -uint64 m_change_stamp
        -unsigned m_pending_delta
        +notify(double, sc_time_unit)
        +notify(const sc_time&)
        +cancel_all()
        +default_event() const sc_event&
        -fire_event()
        +kind() "sc_event_queue"
    }
```

## Key Methods

### `notify()` - Schedule Notification

```cpp
virtual void notify(const sc_time& when);

inline void notify(double when, sc_time_unit base)
{
    notify( sc_time(when, base) );
}
```

Adds a new notification to the queue. The notification will fire after `when` time.

### `cancel_all()` - Cancel All Pending Notifications

Clears all notifications in the queue that have not yet fired.

### `default_event()` - Get Trigger Event

```cpp
const sc_event& default_event() const
{
    return m_e;
}
```

Returns the queue's internal event. Processes can listen to the queue's triggers via `sensitive << event_queue.default_event()`.

## Internal Mechanism

```mermaid
sequenceDiagram
    participant User as User
    participant EQ as sc_event_queue
    participant PPQ as Priority Queue
    participant Kernel as Kernel Scheduler

    User->>EQ: notify(10, SC_NS)
    EQ->>PPQ: Add time=10ns

    User->>EQ: notify(10, SC_NS)
    EQ->>PPQ: Add another time=10ns

    User->>EQ: notify(20, SC_NS)
    EQ->>PPQ: Add time=20ns

    Note over Kernel: === time = 10ns ===
    Kernel->>EQ: fire_event()
    EQ->>EQ: m_e.notify() [first 10ns notification]
    Note over EQ: delta cycle 0

    Kernel->>EQ: fire_event()
    EQ->>EQ: m_e.notify_next_delta() [second 10ns notification]
    Note over EQ: delta cycle 1

    Note over Kernel: === time = 20ns ===
    Kernel->>EQ: fire_event()
    EQ->>EQ: m_e.notify() [20ns notification]
```

### Multiple Notifications at the Same Time

When multiple notifications are scheduled for the same simulation time, the event queue uses delta cycles to distinguish them:
- The first notification fires immediately
- Subsequent notifications fire in the next delta cycle
- This ensures each `notify()` is detected by sensitive processes

## As a Hierarchical Channel

`sc_event_queue` inherits from `sc_module` (not `sc_prim_channel`) because it uses an internal process (`fire_event`) to implement the triggering logic. This makes it a "hierarchical channel" rather than a "primitive channel".

## Port Type

```cpp
typedef sc_port<sc_event_queue_if, 1, SC_ONE_OR_MORE_BOUND> sc_event_queue_port;
```

Provides a convenient port type alias, allowing modules to connect to event queues through ports.

## Use Cases

1. **Interrupt controller**: Multiple interrupt sources may fire simultaneously, each needing to be handled
2. **DMA controller**: Multiple transfer requests scheduled at different times
3. **Network simulation**: Scheduling of multiple packet arrival events
4. **Testbench**: Time-scheduled stimulus signals

## Design Notes

### sc_event vs sc_event_queue

| Property | sc_event | sc_event_queue |
|----------|----------|----------------|
| Multiple notify at same time | Only one kept | Each one kept |
| cancel | Cancels the most recent one | Cancels all |
| Inherits from | Standalone class | sc_module |
| Performance | Faster | Queue overhead |

### Why use a priority queue?

Notifications may be added in arbitrary order but must fire in chronological order. The priority queue (`sc_ppq`) automatically maintains time ordering.

## Related Files

- `sc_interface.h` - `sc_event_queue_if` inherits from `sc_interface`
- `sc_port.h` - `sc_event_queue_port` uses `sc_port`
- `sc_event.h` - Internally uses `sc_event` to trigger
