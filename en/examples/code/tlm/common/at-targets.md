---
title: "AT Targets -- Asynchronous Request Handlers"
linkTitle: "AT Targets"
weight: 5
description: "Implementation analysis of at_target_1_phase, at_target_1_phase_dmi, at_target_2_phase, and at_target_4_phase"
---

## Overview

AT (Approximately-Timed) targets use `nb_transport_fw` to receive requests and `nb_transport_bw` to return results. Different target implementations use different phase protocols, providing a range of timing accuracy options from simple to comprehensive.

### Software Analogy: HTTP Protocol Evolution

| AT Target | Software Analogy | Characteristics |
|-----------|------------------|-----------------|
| `at_target_1_phase` | HTTP/1.0 | Responds immediately upon receiving a request, completed in one step |
| `at_target_2_phase` | HTTP/1.1 | Split into request and response stages |
| `at_target_4_phase` | HTTP/2 | Full request-accept-response-confirm flow |

## Phase Protocol Comparison

```mermaid
sequenceDiagram
    participant I as Initiator
    participant T as Target

    rect rgb(230, 245, 230)
    Note over I,T: 1-Phase (TLM_COMPLETED)
    I->>T: nb_transport_fw(gp, BEGIN_REQ)
    T-->>I: TLM_COMPLETED + annotated delay
    end

    rect rgb(230, 235, 250)
    Note over I,T: 2-Phase (END_REQ + BEGIN_RESP)
    I->>T: nb_transport_fw(gp, BEGIN_REQ)
    T-->>I: TLM_UPDATED(phase=END_REQ)
    Note over T: After PEQ delay...
    T->>I: nb_transport_bw(gp, BEGIN_RESP)
    I->>T: nb_transport_fw(gp, END_RESP)
    T-->>I: TLM_COMPLETED
    end

    rect rgb(250, 235, 230)
    Note over I,T: 4-Phase (full handshake)
    I->>T: nb_transport_fw(gp, BEGIN_REQ)
    T-->>I: TLM_ACCEPTED
    T->>I: nb_transport_bw(gp, END_REQ)
    Note over T: After processing delay...
    T->>I: nb_transport_bw(gp, BEGIN_RESP)
    I->>T: nb_transport_fw(gp, END_RESP)
    T-->>I: TLM_COMPLETED
    end
```

## Common Architecture

All AT targets inherit `tlm_fw_transport_if<>` and implement the following interfaces:

- **`nb_transport_fw`** -- Handles forward requests from the initiator
- **`begin_response_method`** (SC_METHOD) -- Dequeues transactions from PEQ and sends results back via `nb_transport_bw`
- **`m_response_PEQ`** (`peq_with_get`) -- Payload Event Queue for delayed response scheduling
- **`m_target_memory`** (`memory` type) -- Performs the actual memory read/write operations

## at_target_1_phase -- Mixed-Mode Target

**Files**: `include/at_target_1_phase.h`, `src/at_target_1_phase.cpp`

This target supports **two** response modes, switching based on a request counter:

- **First 19 transactions** (`m_request_count % 20 != 0`): Returns `TLM_COMPLETED` directly (1-phase mode)
- **Every 20th transaction**: Returns `TLM_UPDATED` (phase = END_REQ), then schedules `BEGIN_RESP` (2-phase mode)

### Workflow

```mermaid
flowchart TD
    REQ["Receive BEGIN_REQ"] --> CHECK{"request_count % 20?"}
    CHECK -- "!= 0<br/>(majority)" --> FAST["1-Phase fast path"]
    FAST --> OP1["memory.operation(gp, delay)"]
    OP1 --> COMP["Return TLM_COMPLETED<br/>delay += accept_delay"]

    CHECK -- "== 0<br/>(every 20th)" --> SLOW["2-Phase delayed path"]
    SLOW --> DELAY["memory.get_delay(gp, delay)"]
    DELAY --> PEQ["m_response_PEQ.notify(gp, delay)"]
    PEQ --> UPDATE["Return TLM_UPDATED<br/>phase = END_REQ"]
    PEQ --> BRM["begin_response_method triggered"]
    BRM --> OP2["memory.operation(gp, delay)"]
    OP2 --> BW["nb_transport_bw(gp, BEGIN_RESP)"]
```

### Why Mix Two Modes?

This design demonstrates an important TLM concept: **an initiator must be able to handle different response modes from the same target**. In real hardware, some operations may complete immediately (e.g., cache hit), while others require multiple steps (e.g., memory access after a cache miss).

## at_target_1_phase_dmi -- 1-Phase + DMI

**Files**: `include/at_target_1_phase_dmi.h`, `src/at_target_1_phase_dmi.cpp`

Same logic as `at_target_1_phase`, but with additional DMI support. The header file actually reuses `at_target_1_phase.h` (both share the same header guard: `__AT_TARGET_1_PHASE_H__`).

The `.cpp` implementation is identical to `at_target_1_phase.cpp`, existing so that different examples can link against different `.o` files.

## at_target_2_phase -- Standard Two-Phase Target

**Files**: `include/at_target_2_phase.h`, `src/at_target_2_phase.cpp`

All requests follow the **2-phase path** (unlike `at_target_1_phase` which has a fast path).

### Workflow

```mermaid
sequenceDiagram
    participant I as Initiator
    participant T as at_target_2_phase

    I->>T: nb_transport_fw(gp, BEGIN_REQ, delay)
    Note over T: Calculate delay<br/>memory.get_delay(gp, delay)
    Note over T: Enqueue to PEQ<br/>m_response_PEQ.notify(gp, delay)
    T-->>I: TLM_UPDATED(phase=END_REQ, delay=accept_delay)

    Note over T: PEQ delay expires...
    Note over T: begin_response_method triggered
    Note over T: memory.operation(gp, delay)
    T->>I: nb_transport_bw(gp, BEGIN_RESP, 0)

    alt TLM_COMPLETED
        I-->>T: TLM_COMPLETED(delay)
        Note over T: next_trigger(delay)
    else TLM_ACCEPTED
        I-->>T: TLM_ACCEPTED
        Note over T: next_trigger(m_end_resp_rcvd_event)
        I->>T: nb_transport_fw(gp, END_RESP)
        Note over T: m_end_resp_rcvd_event.notify()
        T-->>I: TLM_COMPLETED
    end
```

### Key Differences

Compared to `at_target_1_phase`:

- **No** fast path (all requests go through PEQ)
- `nb_transport_fw` on `BEGIN_REQ` **does not execute** the memory operation (only calculates the delay)
- Memory operation is deferred to `begin_response_method`

## at_target_4_phase -- Full Four-Phase Target

**Files**: `include/at_target_4_phase.h`, `src/at_target_4_phase.cpp`

Implements the full 4-phase protocol, providing the most precise timing model.

### Workflow

```mermaid
sequenceDiagram
    participant I as Initiator
    participant T as at_target_4_phase

    I->>T: nb_transport_fw(gp, BEGIN_REQ, delay)
    Note over T: Enqueue to m_end_request_PEQ<br/>delay = delay + accept_delay
    T-->>I: TLM_ACCEPTED

    Note over T: PEQ delay expires...
    Note over T: end_request_method triggered
    Note over T: Calculate memory delay
    Note over T: Enqueue to m_response_PEQ
    T->>I: nb_transport_bw(gp, END_REQ, 0)
    I-->>T: TLM_ACCEPTED

    Note over T: m_response_PEQ delay expires...
    Note over T: begin_response_method triggered
    Note over T: memory.operation(gp, delay)
    T->>I: nb_transport_bw(gp, BEGIN_RESP, 0)

    alt TLM_COMPLETED
        I-->>T: TLM_COMPLETED(delay)
        Note over T: next_trigger(delay)
    else TLM_ACCEPTED
        I-->>T: TLM_ACCEPTED
        Note over T: Wait for END_RESP
        I->>T: nb_transport_fw(gp, END_RESP, delay)
        Note over T: m_end_resp_rcvd_event.notify(delay)
        T-->>I: TLM_COMPLETED
    end
```

### Architectural Features

```mermaid
graph TD
    subgraph at_target_4_phase
        NB["nb_transport_fw<br/>(receive BEGIN_REQ)"]
        ER_PEQ["m_end_request_PEQ"]
        ER["end_request_method<br/>(SC_METHOD)"]
        R_PEQ["m_response_PEQ"]
        BR["begin_response_method<br/>(SC_METHOD)"]
        MEM["m_target_memory"]
    end

    NB -- "1. BEGIN_REQ<br/>enqueue to PEQ" --> ER_PEQ
    ER_PEQ -- "2. Delay expires" --> ER
    ER -- "3. Calculate delay<br/>enqueue to PEQ" --> R_PEQ
    ER -- "3. nb_transport_bw<br/>END_REQ" --> Initiator
    R_PEQ -- "4. Delay expires" --> BR
    BR -- "5. Execute operation" --> MEM
    BR -- "5. nb_transport_bw<br/>BEGIN_RESP" --> Initiator
```

Key differences from 2-phase:

| Aspect | 2-phase | 4-phase |
|--------|---------|---------|
| Number of PEQs | 1 (`m_response_PEQ`) | 2 (`m_end_request_PEQ` + `m_response_PEQ`) |
| Number of SC_METHODs | 1 (`begin_response_method`) | 2 (`end_request_method` + `begin_response_method`) |
| BEGIN_REQ return | `TLM_UPDATED` (phase = END_REQ) | `TLM_ACCEPTED` (END_REQ sent later) |
| END_REQ delivery | Embedded in `nb_transport_fw` return value | Separate `nb_transport_bw` call |
| Timing separation | request/response: two timing points | accept/end_request/begin_response/end_response: four timing points |

## Overall Comparison of Four AT Targets

| Feature | 1-phase | 1-phase DMI | 2-phase | 4-phase |
|---------|---------|-------------|---------|---------|
| Phase count | Mixed 1/2 | Mixed 1/2 | Fixed 2 | Fixed 4 |
| Number of PEQs | 1 | 1 | 1 | 2 |
| Number of SC_METHODs | 1 | 1 | 1 | 2 |
| DMI support | No | Yes (stub) | No | No |
| Fast path | Yes (19/20 txns) | Yes (19/20 txns) | No | No |
| Timing accuracy | Low-Medium | Low-Medium | Medium | High |
| Extension support | No | No | No | Yes (optional) |
| Use case | Mixed-mode testing | DMI + AT testing | Standard AT verification | Precise timing model |

### Extension Support (at_target_4_phase)

`at_target_4_phase` optionally supports the `extension_initiator_id` extension at compile time:

```cpp
#ifdef USING_EXTENSION_OPTIONAL
extension_initiator_id *extension_pointer;
gp.get_extension(extension_pointer);
if (extension_pointer) {
    // Can read extension_pointer->m_initiator_id
}
#endif
```

This demonstrates the TLM generic payload extension mechanism -- similar to HTTP headers, allowing custom information to be attached to the standard payload.
