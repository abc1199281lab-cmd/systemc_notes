# Simple Bus -- Master Modules

## Overview

This example contains three master modules, each demonstrating a different bus access pattern. All three are `SC_MODULE` instances using `SC_THREAD` processes (which can call `wait()`).

**Software analogy:**

| Master | Software Equivalent |
|---|---|
| Blocking | Synchronous API call: `response = httpClient.post(data)` |
| Non-blocking | Asynchronous polling: `jobId = queue.submit(task); while (!queue.isDone(jobId)) sleep()` |
| Direct | In-process function call: `value = cache.get(key)` |

---

## Comparison Table

| Aspect | `master_blocking` | `master_non_blocking` | `master_direct` |
|---|---|---|---|
| Bus interface | `simple_bus_blocking_if` | `simple_bus_non_blocking_if` | `simple_bus_direct_if` |
| Priority | 4 (lower priority) | 3 (higher priority) | N/A (no arbitration) |
| Data granularity | 16-word burst | Single word | Single word |
| Target address | `0x4c` (fast mem) | `0x38..0xB8` (cyclic) | `0x78` (fast mem, read-only) |
| Timeout | 300 ns | 20 ns | 100 ns |
| Lock | false | false | N/A |
| Role | Batch data processor | Incremental writer | Monitor/debugger |

---

## File: `simple_bus_master_blocking.h` / `.cpp`

### Software Analogy

This master is like a **batch ETL job**: read a large chunk of data, process it, write it back, then sleep.

### Structure

```cpp
SC_MODULE(simple_bus_master_blocking) {
    sc_in_clk clock;
    sc_port<simple_bus_blocking_if> bus_port;
    // Set via constructor: priority, address, lock, timeout
};
```

### Behavior: `main_action()` (SC_THREAD)

```mermaid
flowchart TD
    A["wait() -- wait for next posedge"] --> B["burst_read(priority=4, addr=0x4c, len=16)<br/>Read 16 words from memory"]
    B --> C{"status == ERROR?"}
    C -->|Yes| D["Print error"]
    C -->|No| E["Loop: mydata[i] += i<br/>wait() between iterations<br/>(simulate computation time)"]
    D --> E
    E --> F["burst_write(priority=4, addr=0x4c, len=16)<br/>Write modified data back"]
    F --> G{"status == ERROR?"}
    G -->|Yes| H["Print error"]
    G -->|No| I["wait(300 ns) -- sleep"]
    H --> I
    I --> A
```

### Key Points

- **Burst length:** 16 words (0x10), so reads addresses `0x4C` to `0x8C`. Note this **crosses** the boundary between fast memory (`0x00-0x7F`) and slow memory (`0x80-0xFF`).
- **Computation simulation:** The `for` loop with `wait()` simulates the CPU spending 16 clock cycles processing data.
- **Priority = 4:** Lower than the non-blocking master (priority 3), meaning the non-blocking master can **preempt** this master's burst transfer.
- **Blocks during transfer:** `burst_read()` and `burst_write()` do not return until all 16 words are transferred -- the SC_THREAD suspends internally via `wait(transfer_done)`.

---

## File: `simple_bus_master_non_blocking.h` / `.cpp`

### Software Analogy

This master is like an **asynchronous microservice client**: submit a request then busy-poll until the result is ready.

### Structure

```cpp
SC_MODULE(simple_bus_master_non_blocking) {
    sc_in_clk clock;
    sc_port<simple_bus_non_blocking_if> bus_port;
    // Set via constructor: priority, start_address, lock, timeout
};
```

### Behavior: `main_action()` (SC_THREAD)

```mermaid
flowchart TD
    A["wait() -- initial synchronization"] --> B["bus_port->read(priority=3, &mydata, addr)"]
    B --> C["Poll: get_status(3)"]
    C --> D{"OK or ERROR?"}
    D -->|"WAIT/REQUEST"| E["wait() -- wait for next posedge"]
    E --> C
    D -->|ERROR| F["Print error"]
    D -->|OK| G["mydata += cnt; cnt++"]
    F --> G
    G --> H["bus_port->write(priority=3, &mydata, addr)"]
    H --> I["Poll: get_status(3)"]
    I --> J{"OK or ERROR?"}
    J -->|"WAIT/REQUEST"| K["wait() -- wait for next posedge"]
    K --> I
    J -->|OK or ERROR| L["wait(20 ns) -- sleep"]
    L --> M["wait() -- wait for next posedge"]
    M --> N["addr += 4"]
    N --> O{"addr > start + 0x80?"}
    O -->|Yes| P["addr = start; cnt = 0"]
    O -->|No| B
    P --> B
```

### Key Points

- **Priority = 3:** Higher than the blocking master, so it can preempt burst transfers.
- **Polling pattern:** After `read()` returns (immediately), the master enters a `while` loop checking `get_status()` every clock cycle. This is the non-blocking pattern -- the master is responsible for detecting completion.
- **Address scan:** Starts at `0x38`, increments by 4 each iteration, wraps back after reaching `0x38 + 0x80 = 0xB8`. Covers both fast and slow memory regions.
- **Read-modify-write:** Each iteration reads one word, adds a counter value, and writes it back.

### Blocking vs. Non-blocking: What's the Real Difference?

Both masters use `SC_THREAD` and both call `wait()`. The difference is **where the wait logic lives**:

- **Blocking:** `wait()` is hidden inside `burst_read()` -- the master just calls the function and gets the result.
- **Non-blocking:** The master explicitly polls with a `while` loop. It has **full control** over what to do between each check (though in this example it just waits).

In software terms, this is the difference between:
```python
# Blocking
result = requests.get(url)  # blocks until response

# Non-blocking
future = session.get(url)   # returns immediately
while not future.done():
    time.sleep(0.01)         # explicit polling
result = future.result()
```

---

## File: `simple_bus_master_direct.h` / `.cpp`

### Software Analogy

This master is a **read-only monitoring dashboard** -- periodically samples memory values and prints them, without going through the bus protocol.

### Structure

```cpp
SC_MODULE(simple_bus_master_direct) {
    sc_in_clk clock;
    sc_port<simple_bus_direct_if> bus_port;
    // Set via constructor: address, timeout, verbose
};
```

### Behavior: `main_action()` (SC_THREAD)

```mermaid
flowchart TD
    A["direct_read(&mydata[0], 0x78)"] --> B["direct_read(&mydata[1], 0x7C)"]
    B --> C["direct_read(&mydata[2], 0x80)"]
    C --> D["direct_read(&mydata[3], 0x84)"]
    D --> E["Print: mem[0x78:0x87] = (val, val, val, val)"]
    E --> F["wait(100 ns)"]
    F --> A
```

### Key Points

- **No priority:** Direct access completely bypasses the arbiter.
- **Reads 4 words:** Addresses `0x78, 0x7C, 0x80, 0x84`. Note `0x78-0x7C` is in fast memory, `0x80-0x84` is in slow memory -- but direct access ignores wait states.
- **Instant execution:** All 4 reads complete in the same simulation time step (no `wait()` between them).
- **Monitor role:** This master never writes -- purely observational. Useful for debugging what other masters are doing to memory.
- **No clock sensitivity in constructor:** Unlike the other masters, `SC_THREAD(main_action)` has no `sensitive << clock.pos()`. The thread runs freely, using `wait(timeout)` to control its own timing.

---

## How Three Masters Connect to the Same Bus

All three masters connect to the **same `simple_bus` instance**, but through different interface perspectives:

```mermaid
graph LR
    subgraph "One simple_bus Object"
        BI["blocking_if perspective"]
        NBI["non_blocking_if perspective"]
        DI["direct_if perspective"]
    end

    MB["master_blocking<br/>sc_port&lt;blocking_if&gt;"] -->|"bus_port(*bus)"| BI
    MNB["master_non_blocking<br/>sc_port&lt;non_blocking_if&gt;"] -->|"bus_port(*bus)"| NBI
    MD["master_direct<br/>sc_port&lt;direct_if&gt;"] -->|"bus_port(*bus)"| DI
```

The binding `master_b->bus_port(*bus)` works because `simple_bus` inherits from `simple_bus_blocking_if`. C++ resolves the correct interface perspective at compile time. Each master only sees the methods defined in its specific interface -- the compiler prevents the direct master from accidentally calling `burst_read()`.
