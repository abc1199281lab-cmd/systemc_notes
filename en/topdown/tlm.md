# Transaction-Level Modeling (TLM)

## Everyday Analogy: A Courier Logistics System

Imagine an entire courier logistics system:

- **RTL-level modeling** = Tracking the exact location of every package at every second -- very precise but extremely laborious
- **TLM modeling** = Only tracking "package shipped from Warehouse A, arrives at Warehouse B in 3 days" -- fast and precise enough
- **Generic Payload** = A standardized package -- no matter what's inside, there's a uniform shipping label on the outside
- **Socket** = The warehouse's shipping and receiving windows -- "shipping window" (initiator) and "receiving window" (target)
- **Transaction** = One package delivery -- the complete process from sending to receiving
- **Temporal Decoupling** = The courier picks up multiple packages at once -- no need to make a separate trip for each package

In the early stages of chip design, you don't need to know the timing of every wire;
you only need to know "the CPU read memory address 0x1000, got data 0xDEADBEEF, and it took 100ns."

---

## What Is TLM? Why Do We Need It?

### The Problem: RTL Is Too Slow

```mermaid
graph TD
    subgraph "RTL-Level Simulation"
        R1["Simulates every clock cycle"]
        R2["Tracks every wire"]
        R3["Very accurate"]
        R4["Very slow<br/>An SoC may take hours to simulate"]
    end

    subgraph "TLM Simulation"
        T1["Only simulates transactions (read/write)"]
        T2["Ignores signal-level details"]
        T3["Accurate enough"]
        T4["Very fast<br/>100~1000x faster"]
    end

    R4 -->|"Not fast enough!<br/>The software team can't wait"| T4

    style R4 fill:#ffcdd2
    style T4 fill:#c8e6c9
```

### TLM Use Cases

```mermaid
flowchart TD
    ARCH["Architecture Exploration<br/>Compare different design options"] --> TLM_USE["Use TLM"]
    SW["Software Development<br/>Start writing drivers<br/>before hardware is ready"] --> TLM_USE
    PERF["Performance Analysis<br/>Bandwidth, latency, contention"] --> TLM_USE
    VERIFY["Functional Verification<br/>Is the overall behavior correct?"] --> TLM_USE

    style TLM_USE fill:#c8e6c9
```

---

## TLM 1.0 vs TLM 2.0

```mermaid
graph LR
    subgraph "TLM 1.0"
        T1_PUT["put / get interfaces"]
        T1_FIFO["tlm_fifo"]
        T1_ANALYSIS["tlm_analysis_port"]
        T1_DESC["Simpler<br/>FIFO-like transaction passing"]
    end

    subgraph "TLM 2.0"
        T2_GP["Generic Payload"]
        T2_SOCKET["Socket (initiator/target)"]
        T2_PHASE["Phase (BEGIN_REQ/END_REQ/...)"]
        T2_BT["Blocking Transport"]
        T2_NBT["Non-blocking Transport"]
        T2_DESC["More complete<br/>Supports timing accuracy control"]
    end

    T1_DESC -->|"Evolution"| T2_DESC

    style T1_DESC fill:#e3f2fd
    style T2_DESC fill:#c8e6c9
```

| Feature | TLM 1.0 | TLM 2.0 |
|---------|---------|---------|
| Main Interfaces | put/get/peek | b_transport / nb_transport_fw/bw |
| Data Format | Arbitrary types | Generic Payload (standardized) |
| Timing Model | None | Yes (AT/LT) |
| Use Cases | Simple data passing | Full bus modeling |
| Standardization | Basic | IEEE 1666-2011 |

---

## Generic Payload

Generic Payload is the standardized transaction format defined by TLM 2.0,
like an "international shipping label" -- no matter what you ship or where it goes, the format is the same.

```mermaid
classDiagram
    class tlm_generic_payload {
        -m_command : tlm_command
        -m_address : uint64
        -m_data : unsigned char*
        -m_length : unsigned int
        -m_response_status : tlm_response_status
        -m_byte_enable : unsigned char*
        -m_byte_enable_length : unsigned int
        -m_streaming_width : unsigned int
        -m_extensions : vector
        +set_command(cmd)
        +set_address(addr)
        +set_data_ptr(data)
        +set_data_length(len)
        +set_response_status(status)
        +get_command() tlm_command
        +get_address() uint64
        +get_data_ptr() unsigned char*
    }

    class tlm_command {
        <<enumeration>>
        TLM_READ_COMMAND
        TLM_WRITE_COMMAND
        TLM_IGNORE_COMMAND
    }

    class tlm_response_status {
        <<enumeration>>
        TLM_OK_RESPONSE
        TLM_INCOMPLETE_RESPONSE
        TLM_ADDRESS_ERROR_RESPONSE
        TLM_GENERIC_ERROR_RESPONSE
    }

    tlm_generic_payload --> tlm_command
    tlm_generic_payload --> tlm_response_status
```

### Common Fields Explained

```mermaid
flowchart TD
    GP["Generic Payload"]

    CMD["Command<br/>READ or WRITE"]
    ADDR["Address<br/>Target address (e.g., 0x1000)"]
    DATA["Data Pointer<br/>Pointer to the data"]
    LEN["Data Length<br/>Data length (bytes)"]
    RESP["Response Status<br/>Whether the transaction succeeded"]
    BE["Byte Enable<br/>Which bytes are valid"]
    SW["Streaming Width<br/>Streaming mode width"]

    GP --> CMD
    GP --> ADDR
    GP --> DATA
    GP --> LEN
    GP --> RESP
    GP --> BE
    GP --> SW

    style CMD fill:#e3f2fd
    style ADDR fill:#e3f2fd
    style DATA fill:#e3f2fd
    style LEN fill:#e3f2fd
    style RESP fill:#c8e6c9
    style BE fill:#fff3e0
    style SW fill:#fff3e0
```

### Usage Example

```cpp
// 建立一個讀取交易
tlm::tlm_generic_payload trans;
unsigned char data[4];

trans.set_command(tlm::TLM_READ_COMMAND);
trans.set_address(0x1000);
trans.set_data_ptr(data);
trans.set_data_length(4);
trans.set_response_status(tlm::TLM_INCOMPLETE_RESPONSE);

// 發送交易
sc_time delay = SC_ZERO_TIME;
socket->b_transport(trans, delay);

// 檢查結果
if (trans.get_response_status() == tlm::TLM_OK_RESPONSE) {
    // data[] 現在包含從 0x1000 讀到的 4 個位元組
}
```

---

## Socket: Initiator and Target

### Basic Concept

```mermaid
flowchart LR
    subgraph "Initiator Module<br/>(CPU)"
        I_PROC["Process"]
        I_SOCK["initiator_socket"]
    end

    subgraph "Target Module<br/>(Memory)"
        T_SOCK["target_socket"]
        T_IMPL["b_transport implementation"]
    end

    I_PROC -->|"calls b_transport"| I_SOCK
    I_SOCK -->|"forward path<br/>(request)"| T_SOCK
    T_SOCK -->|"calls"| T_IMPL
    T_IMPL -->|"backward path<br/>(response)"| I_SOCK

    style I_SOCK fill:#c8e6c9
    style T_SOCK fill:#fff3e0
```

### Socket Class Hierarchy

```mermaid
classDiagram
    class tlm_fw_transport_if~TYPES~ {
        <<abstract>>
        +b_transport(trans, delay)*
        +nb_transport_fw(trans, phase, delay)*
        +get_direct_mem_ptr(trans, dmi_data)*
        +transport_dbg(trans)*
    }

    class tlm_bw_transport_if~TYPES~ {
        <<abstract>>
        +nb_transport_bw(trans, phase, delay)*
        +invalidate_direct_mem_ptr(start, end)*
    }

    class tlm_initiator_socket~BUSWIDTH~ {
        +bind(target_socket)
        +operator->() fw_if
    }

    class tlm_target_socket~BUSWIDTH~ {
        +bind(initiator_socket)
    }

    tlm_initiator_socket --> tlm_fw_transport_if : uses
    tlm_initiator_socket --> tlm_bw_transport_if : provides
    tlm_target_socket --> tlm_fw_transport_if : provides
    tlm_target_socket --> tlm_bw_transport_if : uses
```

### Binding

```cpp
// 直接綁定
initiator.socket.bind(target.socket);

// 或用運算子
initiator.socket(target.socket);
```

```mermaid
flowchart LR
    subgraph "System Architecture Example"
        CPU["CPU<br/>initiator_socket"]
        BUS["Bus<br/>target + initiator"]
        MEM["Memory<br/>target_socket"]
        UART["UART<br/>target_socket"]
    end

    CPU -->|bind| BUS
    BUS -->|bind| MEM
    BUS -->|bind| UART

    style CPU fill:#c8e6c9
    style BUS fill:#e3f2fd
    style MEM fill:#fff3e0
    style UART fill:#fff3e0
```

---

## Two Transport Modes

### Blocking Transport

```cpp
void b_transport(tlm::tlm_generic_payload& trans, sc_time& delay);
```

The entire transaction completes in a single function call, like a phone call --
you dial, wait for the connection, have a conversation, and hang up, all in one call.

```mermaid
sequenceDiagram
    participant I as Initiator
    participant T as Target

    I->>T: b_transport(trans, delay)
    Note over T: Process transaction<br/>(read/write memory)
    Note over T: Set response_status
    Note over T: Accumulate delay
    T-->>I: Return
    Note over I: Check result<br/>wait(delay)
```

### Non-blocking Transport

```cpp
tlm_sync_enum nb_transport_fw(tlm_generic_payload& trans,
                               tlm_phase& phase,
                               sc_time& delay);
```

The transaction completes in multiple phases, like sending a package --
order placed, picked up, in transit, out for delivery, signed for -- each phase is independent.

```mermaid
sequenceDiagram
    participant I as Initiator
    participant T as Target

    I->>T: nb_transport_fw(trans, BEGIN_REQ, delay)
    Note over T: Request received

    T->>I: nb_transport_bw(trans, END_REQ, delay)
    Note over I: Request accepted

    Note over T: Processing...

    T->>I: nb_transport_bw(trans, BEGIN_RESP, delay)
    Note over I: Response received

    I->>T: nb_transport_fw(trans, END_RESP, delay)
    Note over T: Response acknowledged
```

---

## Loosely-Timed vs Approximately-Timed

### Loosely-Timed (LT) -- Coarse Timing

```mermaid
flowchart TD
    LT["Loosely-Timed (LT)"]
    LT1["Uses b_transport"]
    LT2["Timing is less precise"]
    LT3["Fastest simulation speed"]
    LT4["Suitable for software development<br/>and early exploration"]
    LT5["Supports temporal decoupling"]

    LT --> LT1
    LT --> LT2
    LT --> LT3
    LT --> LT4
    LT --> LT5

    style LT fill:#c8e6c9
```

### Approximately-Timed (AT) -- Approximate Timing

```mermaid
flowchart TD
    AT["Approximately-Timed (AT)"]
    AT1["Uses nb_transport"]
    AT2["Timing is more precise"]
    AT3["Slower simulation speed"]
    AT4["Suitable for performance analysis"]
    AT5["Can model pipelines and parallelism"]

    AT --> AT1
    AT --> AT2
    AT --> AT3
    AT --> AT4
    AT --> AT5

    style AT fill:#fff3e0
```

### Accuracy vs Speed Trade-off

```mermaid
graph LR
    FAST["Simulation Speed"] -->|"High"| LT["Loosely-Timed<br/>(b_transport)"]
    FAST -->|"Medium"| AT["Approximately-Timed<br/>(nb_transport)"]
    FAST -->|"Low"| RTL["RTL<br/>(signal-level)"]

    ACCURATE["Timing Accuracy"] -->|"Low"| LT
    ACCURATE -->|"Medium"| AT
    ACCURATE -->|"High"| RTL

    style LT fill:#c8e6c9
    style AT fill:#fff3e0
    style RTL fill:#ffcdd2
```

---

## Phase (Transaction Phases)

TLM 2.0 defines four basic phases:

```mermaid
stateDiagram-v2
    [*] --> BEGIN_REQ: Initiator sends request
    BEGIN_REQ --> END_REQ: Target accepts request
    END_REQ --> BEGIN_RESP: Target begins response
    BEGIN_RESP --> END_RESP: Initiator accepts response
    END_RESP --> [*]: Transaction complete

    state BEGIN_REQ {
        [*] --> RequestStart
        RequestStart --> AddressAndCommandInTransit
    }

    state END_REQ {
        [*] --> RequestEnd
        RequestEnd --> TargetCanProcess
    }

    state BEGIN_RESP {
        [*] --> ResponseStart
        ResponseStart --> DataInTransit
    }

    state END_RESP {
        [*] --> ResponseEnd
        ResponseEnd --> InitiatorReceivedData
    }
```

### Phase Mapped to Bus Behavior

```mermaid
sequenceDiagram
    participant CPU as CPU (Initiator)
    participant BUS as Bus
    participant MEM as Memory (Target)

    Note over CPU, MEM: BEGIN_REQ
    CPU->>BUS: Address + command placed on bus

    Note over CPU, MEM: END_REQ
    BUS->>MEM: Arbitration complete, request delivered

    Note over CPU, MEM: BEGIN_RESP
    MEM->>BUS: Data placed on bus

    Note over CPU, MEM: END_RESP
    BUS->>CPU: Data delivered to CPU
```

---

## Temporal Decoupling and Quantum

### The Problem: Synchronizing Too Often

In normal simulation, every transaction must synchronize with the simulation engine (by calling `wait()`),
which severely slows down simulation speed.

### The Solution: Let the Process Run Ahead of Simulation Time

```mermaid
flowchart TD
    subgraph "Without Temporal Decoupling"
        N1["Transaction 1 -> wait(10ns)"]
        N2["Transaction 2 -> wait(10ns)"]
        N3["Transaction 3 -> wait(10ns)"]
        N4["Must synchronize with the engine<br/>every time -- slow!"]
        N1 --> N2 --> N3 --> N4
    end

    subgraph "With Temporal Decoupling"
        T1["Transaction 1 -> local_time += 10ns"]
        T2["Transaction 2 -> local_time += 10ns"]
        T3["Transaction 3 -> local_time += 10ns"]
        T4["local_time > quantum?<br/>Only then synchronize with the engine"]
        T1 --> T2 --> T3 --> T4
    end

    style N4 fill:#ffcdd2
    style T4 fill:#c8e6c9
```

### Quantum Keeper

```mermaid
classDiagram
    class tlm_quantumkeeper {
        -m_local_time : sc_time
        -m_next_sync_point : sc_time
        +set(local_time)
        +inc(delta)
        +get_local_time() sc_time
        +get_current_time() sc_time
        +need_sync() bool
        +sync()
        +reset()
    }

    class tlm_global_quantum {
        -m_global_quantum : sc_time
        +set(quantum)
        +get() sc_time
        +compute_local_quantum() sc_time
        +instance() tlm_global_quantum
    }

    tlm_quantumkeeper --> tlm_global_quantum : reads
```

### Usage Example

```cpp
// 設定 global quantum (例如 1 微秒)
tlm::tlm_global_quantum::instance().set(sc_time(1, SC_US));

// 在 initiator 中
tlm_utils::tlm_quantumkeeper qk;
qk.reset();

void run() {
    while (true) {
        // 執行交易
        sc_time delay = SC_ZERO_TIME;
        socket->b_transport(trans, delay);

        // 累加本地時間
        qk.inc(delay);

        // 檢查是否需要同步
        if (qk.need_sync()) {
            qk.sync();  // wait(local_time)
        }
    }
}
```

```mermaid
sequenceDiagram
    participant I as Initiator
    participant QK as QuantumKeeper
    participant SIM as Simulation Engine

    Note over SIM: Simulation time = 0ns
    Note over QK: quantum = 100ns

    I->>QK: inc(10ns) [Transaction 1]
    Note over QK: local_time = 10ns

    I->>QK: inc(20ns) [Transaction 2]
    Note over QK: local_time = 30ns

    I->>QK: inc(40ns) [Transaction 3]
    Note over QK: local_time = 70ns

    I->>QK: inc(50ns) [Transaction 4]
    Note over QK: local_time = 120ns > quantum!

    QK->>QK: need_sync() = true
    QK->>SIM: sync() -> wait(120ns)
    Note over SIM: Simulation time = 120ns
    Note over QK: local_time = 0ns
```

---

## DMI (Direct Memory Interface)

DMI allows an initiator to directly access the target's memory,
bypassing the transport interface -- like the courier giving you the key so you can go to the warehouse yourself from now on.

```mermaid
flowchart TD
    subgraph "Normal transport"
        P1["Every read/write requires<br/>calling b_transport"]
        P2["Goes through socket -> target"]
        P3["Slower but fully traceable"]
    end

    subgraph "DMI"
        D1["Request DMI pointer once"]
        D2["Then directly read/write<br/>memory via pointer"]
        D3["Extremely fast! But no tracing"]
    end

    P1 --> P2 --> P3
    D1 --> D2 --> D3

    style P3 fill:#fff3e0
    style D3 fill:#c8e6c9
```

---

## TLM 2.0 Convenience Sockets

`tlm_utils` provides simplified sockets that reduce boilerplate code:

```mermaid
classDiagram
    class simple_initiator_socket {
        +register_nb_transport_bw(callback)
        +register_invalidate_direct_mem_ptr(callback)
    }

    class simple_target_socket {
        +register_b_transport(callback)
        +register_nb_transport_fw(callback)
        +register_get_direct_mem_ptr(callback)
        +register_transport_dbg(callback)
    }

    class multi_passthrough_initiator_socket {
        +register_nb_transport_bw(callback)
        +bind(target)
    }

    class multi_passthrough_target_socket {
        +register_b_transport(callback)
        +register_nb_transport_fw(callback)
    }

    note for simple_target_socket "Most commonly used!<br/>Just register a b_transport<br/>callback and you're done"
```

---

## Complete TLM 2.0 System Example

```mermaid
flowchart TD
    subgraph "SoC Platform Model"
        CPU["CPU<br/>simple_initiator_socket"]
        DMA["DMA<br/>initiator + target"]
        BUS["Interconnect<br/>multi_passthrough<br/>target + initiator"]
        RAM["RAM<br/>simple_target_socket<br/>+ DMI support"]
        UART["UART<br/>simple_target_socket"]
        GPIO["GPIO<br/>simple_target_socket"]
    end

    CPU -->|"0x0000-0xFFFF"| BUS
    DMA -->|"0x0000-0xFFFF"| BUS
    BUS -->|"0x0000-0x7FFF"| RAM
    BUS -->|"0x8000-0x800F"| UART
    BUS -->|"0x8010-0x801F"| GPIO
    CPU -->|"Configure DMA"| DMA

    style CPU fill:#c8e6c9
    style DMA fill:#e3f2fd
    style BUS fill:#fff3e0
    style RAM fill:#fce4ec
    style UART fill:#fce4ec
    style GPIO fill:#fce4ec
```

---

## Related Modules

| Concept | File | Relationship |
|---------|------|--------------|
| Communication Mechanisms | [communication.md](communication.md) | TLM is a higher-level communication abstraction |
| Module Hierarchy | [hierarchy.md](hierarchy.md) | TLM modules are still sc_module |
| Event Mechanism | [events.md](events.md) | Temporal decoupling reduces event synchronization |
| Scheduling Mechanism | [scheduling.md](scheduling.md) | Quantum affects scheduling frequency |

### Corresponding Source Code Documentation

| Source Code Concept | Code Documentation |
|--------------------|--------------------|
| tlm_generic_payload | [doc_v2/code/tlm_core/tlm_2/tlm_generic_payload.md](../code/tlm_core/tlm_2/tlm_generic_payload.md) |
| tlm_phase | [doc_v2/code/tlm_core/tlm_2/tlm_phase.md](../code/tlm_core/tlm_2/tlm_phase.md) |
| tlm_fw_bw_ifs | [doc_v2/code/tlm_core/tlm_2/tlm_fw_bw_ifs.md](../code/tlm_core/tlm_2/tlm_fw_bw_ifs.md) |
| tlm_initiator_socket | [doc_v2/code/tlm_core/tlm_2/tlm_initiator_socket.md](../code/tlm_core/tlm_2/tlm_initiator_socket.md) |
| tlm_target_socket | [doc_v2/code/tlm_core/tlm_2/tlm_target_socket.md](../code/tlm_core/tlm_2/tlm_target_socket.md) |
| tlm_global_quantum | [doc_v2/code/tlm_core/tlm_2/tlm_global_quantum.md](../code/tlm_core/tlm_2/tlm_global_quantum.md) |
| tlm_dmi | [doc_v2/code/tlm_core/tlm_2/tlm_dmi.md](../code/tlm_core/tlm_2/tlm_dmi.md) |
| tlm_quantumkeeper | [doc_v2/code/tlm_utils/tlm_quantumkeeper.md](../code/tlm_utils/tlm_quantumkeeper.md) |
| simple_initiator_socket | [doc_v2/code/tlm_utils/simple_initiator_socket.md](../code/tlm_utils/simple_initiator_socket.md) |
| simple_target_socket | [doc_v2/code/tlm_utils/simple_target_socket.md](../code/tlm_utils/simple_target_socket.md) |
| tlm_analysis | [doc_v2/code/tlm_core/tlm_1/tlm_analysis.md](../code/tlm_core/tlm_1/tlm_analysis.md) |
| tlm_req_rsp | [doc_v2/code/tlm_core/tlm_1/tlm_req_rsp.md](../code/tlm_core/tlm_1/tlm_req_rsp.md) |
| peq_with_cb_and_phase | [doc_v2/code/tlm_utils/peq_with_cb_and_phase.md](../code/tlm_utils/peq_with_cb_and_phase.md) |

---

## Learning Tips

1. **Learn b_transport first, then nb_transport** -- in most cases, LT mode is sufficient
2. **Generic Payload is the core of TLM 2.0** -- understand every field
3. **Socket = Port + Export combined** -- it provides both forward and backward paths
4. **Temporal Decoupling is key to speed** -- without it, TLM's speed advantage is greatly diminished
5. **DMI makes memory access blazing fast** -- very important for memory-intensive simulations
6. **TLM and RTL can be mixed** -- model most of the system with TLM, use RTL only for the parts you care about
7. **simple_target_socket is your best friend** -- it eliminates a lot of boilerplate code
8. **The Extension mechanism makes Generic Payload extensible** -- when you need custom fields, you don't have to modify the payload itself
