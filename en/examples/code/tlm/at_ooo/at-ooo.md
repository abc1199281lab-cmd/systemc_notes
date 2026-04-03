# at_ooo -- Source Code Walkthrough

> **Source Code Path**: `ref/systemc/examples/tlm/at_ooo/`

## Software Analogy Overview

The core concept of this example can be illustrated with a restaurant analogy:

```
Customer A orders a steak (takes 30 minutes)  --> ordered first
Customer B orders a salad (takes 5 minutes)   --> ordered second

In-order restaurant:
  Make steak first (30 min), then salad (5 min) = Customer B waits 35 minutes

Out-of-order restaurant:
  Start both at the same time, salad is served in 5 minutes = Customer B waits only 5 minutes
```

In TLM, `at_target_ooo_2_phase` is this out-of-order restaurant.

## System Architecture

```
example_system_top
  |-- SimpleBusAT<2, 2>       m_bus
  |-- at_target_2_phase       m_at_target_2_phase_1     (ID=201) -- normal target
  |-- at_target_ooo_2_phase   m_at_target_ooo_2_phase_1 (ID=202) -- OOO target
  |-- initiator_top           m_initiator_1              (ID=101)
  |-- initiator_top           m_initiator_2              (ID=102)
```

### OOO Target's Special Delay Settings

```cpp
m_at_target_ooo_2_phase_1(
    "m_at_target_ooo_2_phase_1",
    202,
    "memory_socket_1",
    4*1024,                                // memory size
    4,                                     // memory width
    sc_core::sc_time(20, sc_core::SC_NS),  // accept_delay  (normal target is 10ns)
    sc_core::sc_time(100, sc_core::SC_NS), // read_delay    (normal target is 50ns)
    sc_core::sc_time(60, sc_core::SC_NS)   // write_delay   (normal target is 30ns)
);
```

The OOO target's delays are intentionally set longer (accept: 20ns vs 10ns, read: 100ns vs 50ns), combined with the extra 700ns OOO delay, making out-of-order behavior easier to observe.

## OOO Target Implementation: at_target_ooo_2_phase

### Header File Analysis

`at_target_ooo_2_phase` is nearly identical to `at_target_2_phase`, but with two key additional member variables:

```cpp
sc_core::sc_time    m_peq_delay_time;           // current PEQ delay
sc_core::sc_time    m_delay_for_out_of_order;   // extra OOO delay (700 ns)
```

### nb_transport_fw -- The Core OOO Mechanism

When `BEGIN_REQ` is received:

```cpp
m_target_memory.get_delay(gp, delay_time);  // get base memory delay
delay_time += m_accept_delay;
m_peq_delay_time = delay_time;

bool ooo_flag(false);
// Every other request, add 700ns of extra delay
if (m_request_count++ % 2) {
    m_peq_delay_time += m_delay_for_out_of_order;  // +700ns!
    ooo_flag = true;
}

m_response_PEQ.notify(gp, m_peq_delay_time);  // put into PEQ
```

Software equivalent:

```python
async def handle_request(request, request_number):
    base_delay = calculate_memory_delay(request)

    if request_number % 2 == 1:
        # Odd requests: simulate slow operation
        total_delay = base_delay + 700  # extra 700ns
    else:
        # Even requests: normal speed
        total_delay = base_delay

    # Enqueue into delay queue -- shorter delays get processed first!
    await asyncio.sleep(total_delay)
    send_response(request)
```

### PEQ Scheduling Causes Out-of-Order Behavior

`peq_with_get` is an event queue **sorted by time**. When two transactions have different expiration times, the one that expires first gets processed first:

```
Timeline:
  t=0:  Received Request A (odd, delay = 120ns + 700ns = 820ns)
  t=5:  Received Request B (even, delay = 120ns)

  t=125: Request B expires --> send BEGIN_RESP for B  (B completes first!)
  t=820: Request A expires --> send BEGIN_RESP for A  (A completes later!)
```

This is the essence of OOO: **PEQ sorts by expiration time, not by insertion time**.

### begin_response_method

Exactly the same as `at_target_2_phase`: retrieves expired transactions from the PEQ, executes the memory operation, and calls `nb_transport_bw(BEGIN_RESP)`. The difference is not here, but in the different PEQ delays set in `nb_transport_fw` that cause different expiration times.

## Normal Target vs OOO Target Comparison

| Aspect | at_target_2_phase (ID=201) | at_target_ooo_2_phase (ID=202) |
| --- | --- | --- |
| Protocol | 2-phase | 2-phase |
| accept_delay | 10 ns | 20 ns |
| read_response_delay | 50 ns | 100 ns |
| write_response_delay | 30 ns | 60 ns |
| OOO mechanism | None | Odd requests get +700ns extra delay |
| Response order | Basically in request order | Even requests may respond before odd requests |

## OOO in Real-World Systems

| Scenario | Software Equivalent | Why OOO is needed |
| --- | --- | --- |
| CPU cache miss | `Promise.race([cacheHit, memoryAccess])` | Cache hits should complete immediately without waiting for miss requests |
| Multi-bank memory | Async DB queries (multiple shards) | Different banks have different latencies |
| DMA transfer | Downloading multiple files in parallel | Smaller files should complete first |
| PCIe transaction | Parallel API calls to different services | Fast services should not be blocked by slow services |

## OOO Handling on the Initiator Side

`select_initiator` natively supports OOO because it uses `waiting_bw_path_map` to track the **independent state of each transaction**. Regardless of the order in which responses arrive, it can correctly:

1. Look up the corresponding transaction in `nb_transport_bw`
2. Update that transaction's state
3. Put the completed transaction into `response_fifo`

Software equivalent:

```python
# Each request has an independent Future, completion order doesn't matter
pending = {}  # Dict[request_id, Future]

async def on_response(request_id, data):
    future = pending.pop(request_id)  # find the corresponding Future
    future.set_result(data)            # completes correctly regardless of order
```

## Key Takeaways

| Concept | Description |
| --- | --- |
| **OOO core** | Through different PEQ delays, some transactions complete later than others |
| **m_delay_for_out_of_order** | 700ns extra delay, added every other request |
| **PEQ sorting** | `peq_with_get` sorts by expiration time, causing natural out-of-order completion |
| **Initiator compatibility** | `select_initiator` uses a per-transaction map to track state, natively supporting OOO |
| **Performance significance** | OOO allows fast operations to not be blocked by slow operations, improving system throughput |
