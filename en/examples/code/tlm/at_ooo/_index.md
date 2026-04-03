# at_ooo -- AT Out-of-Order Completion Example

> **Difficulty**: Advanced | **Software Analogy**: `Promise.race()` / async responses arriving in arbitrary order | **Source Code**: `ref/systemc/examples/tlm/at_ooo/`

## Overview

`at_ooo` demonstrates **out-of-order (OOO) transaction completion** in TLM-2.0 AT mode. In this example, requests that are sent first do not necessarily complete first -- just like when you send multiple API requests simultaneously, responses may come back in any order.

### Software Analogy: Promise.all but response order is not guaranteed

```javascript
// Send 3 API requests simultaneously
const p1 = fetch("/api/slow");    // takes 700ms
const p2 = fetch("/api/fast");    // takes 100ms
const p3 = fetch("/api/medium");  // takes 300ms

// Response arrival order: p2 (100ms), p3 (300ms), p1 (700ms)
// Not the sending order: p1, p2, p3
```

Or using Python's `asyncio`:

```python
async def main():
    tasks = [
        fetch_slow(),    # sent 1st, but completes last
        fetch_fast(),    # sent 2nd, but completes first
        fetch_medium(),  # sent 3rd, completes in the middle
    ]
    # asyncio.as_completed returns in completion order
    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(f"Got result: {result}")
```

### Why is OOO needed?

In real hardware systems, different access operations have different latencies:
- **Cache hit**: a few clock cycles
- **Cache miss**: requires accessing main memory, hundreds of clock cycles
- **I/O device access**: can take thousands of clock cycles

If all requests must complete in order (in-order), faster requests behind slower ones must wait for the slow requests to finish, wasting performance. **OOO allows fast requests to complete first**, significantly improving system throughput.

```
In-order (inefficient):
  Request A (slow) ----[============================]
  Request B (fast) --                                --[===]
  Request C (fast) --                                       --[===]
                                                              ^-- all complete

Out-of-order (efficient):
  Request A (slow) ----[============================]
  Request B (fast) ----[===]
  Request C (fast) ----[===]
                                ^-- all complete (faster!)
```

## Architecture Diagram

```mermaid
graph LR
    subgraph initiator_top_1["initiator_top (ID=101)"]
        TG1[traffic_generator] -->|request_fifo| I1[select_initiator]
        I1 -->|response_fifo| TG1
    end

    subgraph initiator_top_2["initiator_top (ID=102)"]
        TG2[traffic_generator] -->|request_fifo| I2[select_initiator]
        I2 -->|response_fifo| TG2
    end

    subgraph SimpleBusAT["SimpleBusAT (2x2)"]
        BUS[Router / Arbiter]
    end

    I1 -->|initiator_socket| BUS
    I2 -->|initiator_socket| BUS
    BUS -->|target_socket| T1["at_target_2_phase (ID=201)<br/>normal target (in-order)"]
    BUS -->|target_socket| T2["at_target_ooo_2_phase (ID=202)<br/>OOO target (out-of-order responses)"]
```

## Out-of-Order Sequence Diagram

```mermaid
sequenceDiagram
    participant I as Initiator
    participant T as at_target_ooo_2_phase

    I->>T: nb_transport_fw(GP_A, BEGIN_REQ) [odd request]
    Note over T: GP_A added to PEQ<br/>delay = normal + 700ns (extra delay!)
    T-->>I: TLM_UPDATED (END_REQ)

    I->>T: nb_transport_fw(GP_B, BEGIN_REQ) [even request]
    Note over T: GP_B added to PEQ<br/>delay = normal (no extra delay)
    T-->>I: TLM_UPDATED (END_REQ)

    Note over T: GP_B's PEQ delay expires first

    T->>I: nb_transport_bw(GP_B, BEGIN_RESP)
    Note over I: GP_B completes first! (even though sent later)
    I-->>T: TLM_COMPLETED

    Note over T: GP_A's PEQ delay expires

    T->>I: nb_transport_bw(GP_A, BEGIN_RESP)
    Note over I: GP_A completes later (even though sent first)
    I-->>T: TLM_COMPLETED
```

## File List

| File | Description | Documentation Link |
| --- | --- | --- |
| `src/at_ooo.cpp` | `sc_main` entry point | [at-ooo.md](at-ooo.md) |
| `src/at_ooo_top.cpp` | System top-level module | [at-ooo.md](at-ooo.md) |
| `src/at_target_ooo_2_phase.cpp` | OOO target implementation | [at-ooo.md](at-ooo.md) |
| `src/initiator_top.cpp` | Initiator top-level module | [at-ooo.md](at-ooo.md) |
| `include/at_ooo_top.h` | Top-level header file | [at-ooo.md](at-ooo.md) |
| `include/at_target_ooo_2_phase.h` | OOO target header file | [at-ooo.md](at-ooo.md) |
| `include/initiator_top.h` | Initiator top-level header file | [at-ooo.md](at-ooo.md) |

## Core Concepts Quick Reference

| TLM Concept | Software Equivalent | Role in This Example |
| --- | --- | --- |
| Out-of-Order | `Promise.race()` / `asyncio.as_completed` | Responses are not returned in request order |
| `m_delay_for_out_of_order` | Artificially added delay (simulating slow operations) | 700ns extra delay makes odd requests slower |
| `m_request_count` | Request counter | Uses `count % 2` to decide whether to add extra delay |
| `peq_with_get` | Priority queue (sorted by time) | Transactions with shorter delays are triggered first |

## Suggested Learning Path

1. It is recommended to first read [at_2_phase](../at_2_phase/_index.md) (the OOO target is based on the 2-phase protocol)
2. Read [at-ooo.md](at-ooo.md) to understand the complete OOO implementation
3. Think about: in your own software projects, which scenarios could benefit from out-of-order processing?
