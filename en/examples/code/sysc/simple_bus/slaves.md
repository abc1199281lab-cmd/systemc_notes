# Simple Bus -- Slave Modules (Memory)

## Overview

This example contains two memory slave modules implementing `simple_bus_slave_if`. Both simulate a contiguous block of RAM but with different response latencies:

| Slave | Address Range | Wait States | Software Analogy |
|---|---|---|---|
| `simple_bus_fast_mem` | `0x00 - 0x7F` | 0 (instant) | In-memory HashMap / Redis cache |
| `simple_bus_slow_mem` | `0x80 - 0xFF` | 1 (configurable) | Disk-backed database / network storage |

**Why two different speeds?** In real hardware, different memory types have vastly different access times -- L1 cache responds in 1 cycle, DRAM takes 50-100 cycles, flash storage takes thousands of cycles. This example simulates that reality in the simplest way.

---

## File: `simple_bus_fast_mem.h`

### Software Analogy

The fast memory slave is like a **HashMap lookup** -- you ask for data and get it instantly in the same clock cycle, no waiting.

### Class Structure

```mermaid
classDiagram
    class simple_bus_fast_mem {
        <<sc_module>>
        -int* MEM
        -unsigned int m_start_address
        -unsigned int m_end_address
        +direct_read(data*, addr) : bool
        +direct_write(data*, addr) : bool
        +read(data*, addr) : status
        +write(data*, addr) : status
        +start_address() : unsigned int
        +end_address() : unsigned int
    }
    simple_bus_slave_if <|-- simple_bus_fast_mem
    sc_module <|-- simple_bus_fast_mem
```

### Key Implementation Details

**Constructor:**
- Allocates an `int[]` array of size `(end_address - start_address + 1) / 4` words
- Initializes all memory to zero
- Asserts the address range is word-aligned (divisible by 4)

**`read()` / `write()`:**
```cpp
inline simple_bus_status simple_bus_fast_mem::read(int *data, unsigned int address) {
    *data = MEM[(address - m_start_address) / 4];
    return SIMPLE_BUS_OK;  // always OK, no wait states
}
```

The address translation `(address - m_start_address) / 4` converts a byte address to a word index. For example, when `m_start_address = 0x00`, address `0x08` maps to `MEM[2]`.

**`direct_read()` / `direct_write()`:**
Delegates directly to `read()` / `write()` and converts status to `bool`.

**No process, no clock port:** Fast memory has no internal state machine -- it responds in the same delta cycle it is called.

---

## File: `simple_bus_slow_mem.h`

### Software Analogy

The slow memory slave is like a **database query with latency**: you submit the query, get "processing..." for N cycles, then finally get the data. It's like calling an API that first returns HTTP 202 (Accepted), then returns 200 (OK) after processing.

### Class Structure

```mermaid
classDiagram
    class simple_bus_slow_mem {
        <<sc_module>>
        +sc_in_clk clock
        -int* MEM
        -unsigned int m_start_address
        -unsigned int m_end_address
        -unsigned int m_nr_wait_states
        -int m_wait_count
        +wait_loop() SC_METHOD
        +direct_read(data*, addr) : bool
        +direct_write(data*, addr) : bool
        +read(data*, addr) : status
        +write(data*, addr) : status
        +start_address() : unsigned int
        +end_address() : unsigned int
    }
    simple_bus_slave_if <|-- simple_bus_slow_mem
    sc_module <|-- simple_bus_slow_mem
```

### Wait State Mechanism

```mermaid
sequenceDiagram
    participant B as Bus (negedge)
    participant S as Slow Memory
    participant W as wait_loop (posedge)

    Note over S: m_wait_count = -1 (idle)

    B->>S: read(data, addr)
    Note over S: m_wait_count = nr_wait_states (= 1)
    S-->>B: SIMPLE_BUS_WAIT

    Note over W: posedge: m_wait_count-- (becomes 0)

    B->>S: read(data, addr) (re-issued)
    Note over S: m_wait_count == 0, transfer data
    S-->>B: SIMPLE_BUS_OK
    Note over S: m_wait_count stays 0 until next new request
```

**`read()` / `write()` state machine:**

```mermaid
stateDiagram-v2
    [*] --> Idle : m_wait_count = -1
    Idle --> Counting : Bus calls read/write<br/>m_wait_count = nr_wait_states<br/>Return WAIT
    Counting --> Counting : m_wait_count > 0<br/>Return WAIT
    Counting --> Ready : m_wait_count == 0<br/>Transfer data<br/>Return OK

    note right of Counting : wait_loop() decrements<br/>m_wait_count on each posedge
```

### Key Implementation Details

**Two-part design:**

1. **`read()` / `write()` methods** (called by bus at negedge):
   - If `m_wait_count < 0`: This is a new request. Set counter to `m_nr_wait_states`, return `SIMPLE_BUS_WAIT`.
   - If `m_wait_count == 0`: Counter has counted down. Perform the actual data transfer, return `SIMPLE_BUS_OK`.
   - Otherwise: Still counting down, return `SIMPLE_BUS_WAIT`.

2. **`wait_loop()` SC_METHOD** (triggered on posedge):
   - Simply decrements if `m_wait_count >= 0`.

**Why does the counter decrement on posedge?** The bus calls `read()`/`write()` at negedge. The counter decrements at posedge (half a cycle later). At the next negedge, the bus re-issues the request and checks if the counter has reached zero. This establishes the wait state timing:

```
posedge  negedge  posedge  negedge
   |        |        |        |
   |   Bus calls  Counter   Bus re-calls
   |   read()    decrements  read()
   |   (WAIT)   (1->0)      (OK, transfer)
```

**`direct_read()` / `direct_write()`:**
Unlike regular `read`/`write`, direct access **completely bypasses wait states** -- reads/writes `MEM[]` directly and returns `true`. This is why the direct master can read slow memory instantly.

---

## Fast vs. Slow: Side-by-Side Comparison

| Aspect | `simple_bus_fast_mem` | `simple_bus_slow_mem` |
|---|---|---|
| Clock port | None | `sc_in_clk clock` |
| Process | None | `SC_METHOD(wait_loop)` on posedge |
| `read()`/`write()` returns | Always `SIMPLE_BUS_OK` | First `SIMPLE_BUS_WAIT` then `SIMPLE_BUS_OK` |
| `direct_read()`/`direct_write()` | Delegates to `read()`/`write()` | Reads `MEM[]` directly (bypasses wait) |
| Wait states | 0 | Configurable (`m_nr_wait_states`) |
| Cycles per word transfer | 1 | 1 + `nr_wait_states` |
| Real-world model | SRAM / L1 cache | DRAM / Flash |

---

## Address Translation Diagram

```
Byte address:  0x00  0x04  0x08  ...  0x7C  0x80  0x84  ...  0xFC
               |---- fast_mem (0x00-0x7F) ----||---- slow_mem (0x80-0xFF) ----|
Word index:    [0]   [1]   [2]  ...  [31]   [0]   [1]  ...  [31]
MEM array:     MEM[0] MEM[1] ...      MEM[0] MEM[1] ...

Formula: MEM[(address - start_address) / 4]
```

Each slave independently translates byte addresses to its own internal array index. The bus decides which slave to route to based on address range; the slave then converts the absolute address to a local offset.
