# communication -- SystemC Communication Subsystem

The `communication` subsystem of SystemC defines all the infrastructure for inter-module communication. It implements four core concepts: **Interface**, **Channel**, **Port**, and **Export**, enabling hardware modules to exchange data and events.

## Overview

Think of a city's postal system:
- **Interface** is like the rules for "sending" and "receiving" mail -- defining what operations are possible
- **Channel** is like the actual mailboxes and postboxes -- implementing data storage and delivery
- **Port** is like the mail slot on each building's entrance -- modules access external channels through it
- **Export** is like a "service window" -- letting a module expose its internal interface to the outside

This architecture follows the "separation of interface and implementation" design principle: modules depend only on interface definitions, not on specific channel implementations, achieving loose coupling.

## Architecture Diagram

```mermaid
graph TB
    subgraph "Core Abstraction Layer"
        IF[sc_interface<br/>Base class of all interfaces]
        PC[sc_prim_channel<br/>Primitive channel base class]
        PB[sc_port_base<br/>Port base class]
        EB[sc_export_base<br/>Export base class]
    end

    subgraph "Signal Interface Layer"
        SIF_IN[sc_signal_in_if&lt;T&gt;<br/>Input interface]
        SIF_WRITE[sc_signal_write_if&lt;T&gt;<br/>Write interface]
        SIF_INOUT[sc_signal_inout_if&lt;T&gt;<br/>Input/output interface]
    end

    subgraph "Signal Channel Layer"
        SIG[sc_signal&lt;T&gt;<br/>Signal channel]
        BUF[sc_buffer&lt;T&gt;<br/>Buffer channel]
        CLK[sc_clock<br/>Clock channel]
    end

    subgraph "Signal Port Layer"
        SP_IN[sc_in&lt;T&gt;<br/>Input port]
        SP_INOUT[sc_inout&lt;T&gt;<br/>Input/output port]
        SP_OUT[sc_out&lt;T&gt;<br/>Output port]
    end

    subgraph "Utilities"
        EF[sc_event_finder<br/>Event finder]
        EQ[sc_event_queue<br/>Event queue]
        WP[sc_writer_policy<br/>Writer policy]
        CID[sc_communication_ids<br/>Error message IDs]
    end

    IF --> PC
    IF --> SIF_IN
    IF --> SIF_WRITE
    SIF_IN --> SIF_INOUT
    SIF_WRITE --> SIF_INOUT
    PC --> SIG
    SIG --> BUF
    SIG --> CLK
    SIF_INOUT -.->|implements| SIG
    PB --> SP_IN
    PB --> SP_INOUT
    SP_INOUT --> SP_OUT
    WP -.->|policy| SIG
    EF -.->|assists| PB
```

## Data Flow Diagram

```mermaid
sequenceDiagram
    participant Module_A as Module A
    participant Port_Out as sc_out (output port)
    participant Signal as sc_signal (signal channel)
    participant Port_In as sc_in (input port)
    participant Module_B as Module B

    Module_A->>Port_Out: write(value)
    Port_Out->>Signal: write(value) via interface
    Note over Signal: Value stored in m_new_val<br/>call request_update()
    Note over Signal: === update phase ===
    Signal->>Signal: update(): m_cur_val = m_new_val
    Signal->>Signal: notify value_changed_event
    Signal->>Port_In: Event triggered
    Port_In->>Module_B: Wake up sensitive process
    Module_B->>Port_In: read()
    Port_In->>Signal: read() via interface
    Signal-->>Module_B: Return m_cur_val
```

## File List

| File | Description |
|------|-------------|
| [sc_interface.md](sc_interface.md) | Abstract base class of all interface classes |
| [sc_port.md](sc_port.md) | Port base class, entry point for modules to access external channels |
| [sc_export.md](sc_export.md) | Export base class, lets modules expose internal interfaces |
| [sc_prim_channel.md](sc_prim_channel.md) | Abstract base class of primitive channels |
| [sc_signal.md](sc_signal.md) | Generic signal channel `sc_signal<T>` |
| [sc_signal_ifs.md](sc_signal_ifs.md) | Signal-related interface definitions |
| [sc_signal_ports.md](sc_signal_ports.md) | Signal-specific port classes (`sc_in`, `sc_inout`, `sc_out`) |
| [sc_writer_policy.md](sc_writer_policy.md) | Signal writer policy, controls multi-writer behavior |
| [sc_buffer.md](sc_buffer.md) | Buffer channel, triggers event on every write |
| [sc_clock.md](sc_clock.md) | Clock channel, generates periodic boolean signal |
| [sc_clock_ports.md](sc_clock_ports.md) | Type aliases for clock-specific ports |
| [sc_event_finder.md](sc_event_finder.md) | Event finder, defers event resolution until binding completes |
| [sc_event_queue.md](sc_event_queue.md) | Event queue, supports multiple pending notifications |
| [sc_communication_ids.md](sc_communication_ids.md) | Error/warning message IDs for the communication subsystem |

## Dependency Diagram

```mermaid
graph LR
    sc_interface --> sc_port
    sc_interface --> sc_export
    sc_interface --> sc_signal_ifs
    sc_interface --> sc_event_queue

    sc_signal_ifs --> sc_signal
    sc_signal_ifs --> sc_signal_ports

    sc_port --> sc_signal_ports
    sc_port --> sc_event_finder

    sc_prim_channel --> sc_signal

    sc_signal --> sc_buffer
    sc_signal --> sc_clock

    sc_writer_policy --> sc_signal
    sc_writer_policy --> sc_signal_ifs

    sc_signal_ports --> sc_clock_ports

    sc_event_finder --> sc_signal_ports

    sc_communication_ids -.->|error messages| sc_port
    sc_communication_ids -.->|error messages| sc_export
    sc_communication_ids -.->|error messages| sc_signal
    sc_communication_ids -.->|error messages| sc_prim_channel
```

## Design Philosophy

### Why separate interfaces, channels, and ports?

In real hardware design, wiring and communication protocols are separate concepts. SystemC follows this philosophy:

1. **Interface** defines "what can be done" (e.g., read, write)
2. **Channel** defines "how it's done" (e.g., using two registers to implement delta cycle updates)
3. **Port** defines "where to access" (bridge connecting modules to channels)
4. **Export** defines "what service is provided" (making a sub-module's interface visible externally)

This separation allows different channel implementations for the same interface, and lets modules communicate without knowing the specific channel type.

## Related Directories

- `sysc/kernel/` - Core simulation engine (events, scheduler)
- `sysc/datatypes/` - Data types (`sc_logic`, `sc_bv`, etc.)
- `sysc/tracing/` - Waveform tracing
