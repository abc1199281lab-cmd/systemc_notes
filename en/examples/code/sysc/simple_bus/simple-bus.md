# Simple Bus -- Bus Channel Implementation

## Overview

`simple_bus` is the core **hierarchical channel** of this example. It is both an `sc_module` (has processes and ports) and implements three `sc_interface` subclasses (blocking, non-blocking, direct). From a software perspective, it is a **connection pool + request dispatcher** -- it receives requests from masters, delegates to the arbiter for scheduling, and forwards operations to the appropriate slave.

**Source files:** `simple_bus.h`, `simple_bus.cpp`

---

## Class Structure

```mermaid
classDiagram
    class simple_bus {
        <<sc_module + channel>>
        +sc_in_clk clock
        +sc_port~simple_bus_arbiter_if~ arbiter_port
        +sc_port~simple_bus_slave_if, 0~ slave_port
        -bool m_verbose
        -simple_bus_request_vec m_requests
        -simple_bus_request* m_current_request
        +main_action() SC_METHOD
        +burst_read() : status
        +burst_write() : status
        +read() : void
        +write() : void
        +get_status() : status
        +direct_read() : bool
        +direct_write() : bool
        -handle_request()
        -get_slave(addr) : slave_if*
        -get_request(priority) : request*
        -get_next_request() : request*
        -clear_locks()
        -end_of_elaboration()
    }
```

### Port Description

| Port | Type | Description |
|------|------|-------------|
| `clock` | `sc_in_clk` | System clock. The bus acts on **negedge (falling edge)**. |
| `arbiter_port` | `sc_port<simple_bus_arbiter_if>` | Single arbiter connection. |
| `slave_port` | `sc_port<simple_bus_slave_if, 0>` | **Multi-port** (`0` means unlimited bindings). Multiple slaves connect to this. |

The `0` template parameter for `slave_port` is noteworthy -- from a software perspective, this is like declaring a dependency as `List<SlaveInterface>` instead of `SlaveInterface`. The bus iterates over all bound slaves to find the correct one by address.

---

## Process: `main_action()`

The bus has one `SC_METHOD` triggered on `clock.neg()` (falling edge):

```mermaid
flowchart TD
    START["Clock negedge triggers main_action()"]
    A{"m_current_request == NULL?"}
    A -->|Yes| B["get_next_request()<br/>Collect pending requests<br/>Call arbiter"]
    A -->|No| C["Slave still processing<br/>(log wait states)"]
    B --> D{"Got a request?"}
    C --> D
    D -->|Yes| E["handle_request()<br/>Dispatch to slave"]
    D -->|No| F["clear_locks()"]
    E --> G{"Slave response?"}
    G -->|SIMPLE_BUS_OK| H["Advance address/data pointers"]
    G -->|SIMPLE_BUS_WAIT| I["Keep m_current_request<br/>(retry next cycle)"]
    G -->|SIMPLE_BUS_ERROR| J["Notify error, clear request"]
    H --> K{"Transfer complete?<br/>(addr > end_addr)"}
    K -->|Yes| L["Notify transfer_done<br/>Clear m_current_request"]
    K -->|No| M["Clear m_current_request<br/>(arbiter re-selects next cycle)"]
```

### Why Clear `m_current_request` Between Burst Words?

After each word in a burst transfer completes (`SIMPLE_BUS_OK` but more data remains), the bus sets `m_current_request = NULL`. This forces the arbiter to re-evaluate all pending requests in the next cycle.

**Software analogy:** Like a cooperative scheduler -- after each quantum (one word transfer), the running task yields, and the scheduler decides if a higher-priority task should run instead. This allows a high-priority non-blocking master to **preempt** an in-progress low-priority burst transfer (unless locked).

---

## Interface Implementations

### Direct Interface (`direct_read` / `direct_write`)

The simplest path -- no arbitration, no request queue:

1. Check address alignment (must be word-aligned, multiple of 4)
2. Call `get_slave(address)` to find the matching slave
3. Forward directly to `slave->direct_read()` or `slave->direct_write()`

This executes in **zero simulation time** -- a function call chain with no `wait()`.

### Non-blocking Interface (`read` / `write` / `get_status`)

1. `get_request(priority)` retrieves (or creates) the request object for this master
2. Fill in request fields (address, data pointer, read/write flag)
3. Set `request->status = SIMPLE_BUS_REQUEST`
4. Return immediately -- the actual transfer happens in `main_action()` at the next negedge

The caller polls with `get_status(priority)`, which simply returns `get_request(priority)->status`.

### Blocking Interface (`burst_read` / `burst_write`)

1. Same setup as non-blocking: fill in the request object
2. But `end_address = start_address + (length-1)*4` for multi-word burst
3. **Key difference:** Calls `wait(request->transfer_done)` -- this suspends the calling SC_THREAD
4. When the bus completes (or errors), `main_action` notifies `transfer_done`
5. The master then executes `wait(clock->posedge_event())` to re-synchronize to the rising edge

```mermaid
sequenceDiagram
    participant M as Blocking Master<br/>(SC_THREAD)
    participant B as Bus<br/>(SC_METHOD, negedge)
    participant A as Arbiter
    participant S as Slave

    Note over M: posedge
    M->>B: burst_read(priority=4, addr=0x4c, len=16)
    Note over M: wait(transfer_done)

    Note over B: negedge
    B->>B: get_next_request()
    B->>A: arbitrate(pending_requests)
    A-->>B: Selected request

    loop For each word in burst
        B->>S: slave->read(data, addr)
        alt Slave returns OK
            B->>B: addr += 4, data++
        else Slave returns WAIT
            Note over B: Retry next negedge
        end
    end

    B->>M: notify(transfer_done)
    Note over M: wait(posedge) to re-synchronize
    M->>M: Continue processing
```

---

## Key Internal Methods

### `get_slave(address)`

Iterates over all bound slaves, returns the one whose address range `[start_address, end_address]` contains the given address. Returns `NULL` if none found.

**Software analogy:** URL route matching -- `/api/users/123` matches route `/api/users/:id`.

### `get_request(priority)`

Searches `m_requests` for a request with the matching priority. Creates a new one if not found. Priority serves simultaneously as the master's unique ID and importance level.

**Software analogy:** Session storage keyed by client ID.

### `get_next_request()`

Collects all requests with status `SIMPLE_BUS_REQUEST` or `SIMPLE_BUS_WAIT`, passes them to `arbiter_port->arbitrate()`, and returns the winner.

### `clear_locks()`

Called when there are no active requests. Downgrades lock states:
- `SIMPLE_BUS_LOCK_GRANTED` -> `SIMPLE_BUS_LOCK_SET` (preserved for next round)
- Other -> `SIMPLE_BUS_LOCK_NO` (released)

### `end_of_elaboration()`

Automatically called by SystemC after all modules are connected and before simulation starts. Checks for **overlapping address spaces** between slaves -- exits with an error if two slaves claim the same address range.

**Software analogy:** Application startup validation -- like checking for duplicate route definitions in a web framework.

---

## Lock Mechanism State Machine

```mermaid
stateDiagram-v2
    [*] --> LOCK_NO : Initial state

    LOCK_NO --> LOCK_SET : Request with lock=true
    LOCK_SET --> LOCK_GRANTED : Arbiter selects this request
    LOCK_GRANTED --> LOCK_SET : Transfer done, clear_locks()
    LOCK_SET --> LOCK_NO : Transfer done but not granted, clear_locks()

    note right of LOCK_SET : Bus reserved but not yet confirmed
    note right of LOCK_GRANTED : Arbiter will prioritize this master
```

The lock mechanism allows a master to **chain multiple bus transactions** without being preempted. It works similarly to a database advisory lock -- the first transaction sets the lock, and if granted, subsequent transactions from the same master are guaranteed bus access.
