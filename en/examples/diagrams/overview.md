# SystemC Official Examples - Global Architecture Overview

## Global Architecture Diagram

```mermaid
flowchart TB
    subgraph SYSC["SystemC Core Examples (sysc/)"]
        direction TB

        subgraph BASIC["Basic Examples"]
            SF["simple_fifo<br/>Producer-Consumer"]
            RSA["rsa<br/>Big Number Arithmetic"]
            SP["simple_perf<br/>Performance Modeling"]
        end

        subgraph DSP["Signal Processing"]
            PIPE["pipe<br/>3-Stage Pipeline"]
            FIR["fir<br/>FIR Filter<br/>(Behavioral+RTL)"]
            FFT["fft<br/>FFT<br/>(Floating-Point+Fixed-Point)"]
        end

        subgraph ARCH["System Architecture"]
            PKT["pkt_switch<br/>Packet Switch"]
            BUS["simple_bus<br/>Bus Arbitration"]
            CPU["risc_cpu<br/>RISC CPU"]
        end

        subgraph VER["Version Features"]
            V21["2.1<br/>Dynamic process<br/>fork-join<br/>barrier"]
            V23["2.3<br/>Handshake Protocol<br/>async event"]
            V24["2.4<br/>in-class init"]
            ASYNC["async_suspend<br/>Cross-Thread Event"]
        end
    end

    subgraph TLM["TLM Transaction-Level Model Examples (tlm/)"]
        direction TB

        COMMON["common/<br/>Shared Component Library<br/>(initiator, target,<br/>memory, bus)"]

        subgraph LT_GROUP["Loosely-Timed (Synchronous)"]
            LT["lt<br/>Basic LT"]
            LT_DMI["lt_dmi<br/>Direct Memory"]
            LT_TD["lt_temporal_decouple<br/>Temporal Decoupling"]
            LT_ME["lt_mixed_endian<br/>Mixed Endianness"]
            LT_EXT["lt_extension_mandatory<br/>Mandatory Extension"]
        end

        subgraph AT_GROUP["Approximately-Timed (Asynchronous)"]
            AT1["at_1_phase<br/>Single Phase"]
            AT2["at_2_phase<br/>Two Phase"]
            AT4["at_4_phase<br/>Four Phase"]
            AT_EO["at_extension_optional<br/>Optional Extension"]
            AT_MT["at_mixed_targets<br/>Mixed Targets"]
            AT_OOO["at_ooo<br/>Out-of-Order Completion"]
        end

        COMMON --> LT_GROUP
        COMMON --> AT_GROUP
    end

    BASIC --> DSP
    BASIC --> ARCH
    DSP --> ARCH
    SYSC --> TLM

    style BASIC fill:#d4edda
    style DSP fill:#fff3cd
    style ARCH fill:#f8d7da
    style VER fill:#cce5ff
    style LT_GROUP fill:#e2d9f3
    style AT_GROUP fill:#fce4ec
```

## Dependency Direction Overview

```mermaid
flowchart LR
    subgraph Learning Path
        direction LR
        A["Introductory Concepts<br/>simple_fifo<br/>pipe"] --> B["Advanced Modeling<br/>fir, fft<br/>pkt_switch"]
        B --> C["System Design<br/>simple_bus<br/>risc_cpu"]
        A --> D["TLM Basics<br/>common<br/>lt"]
        D --> E["TLM Advanced<br/>lt_dmi<br/>at_2_phase<br/>at_4_phase"]
        E --> F["TLM Specialized<br/>at_ooo<br/>extensions"]
    end
```

## File Statistics

| Category | Example Count | Source Files | Documentation Files | Has spec.md |
| --- | --- | --- | --- | --- |
| sysc Basic | 3 | 4 | 11 | 2 |
| sysc Pipeline/DSP | 3 | 39 | 25 | 3 |
| sysc System Architecture | 3 | 58 | 29 | 3 |
| sysc Version Features | 4 | 19 | 19 | 0 |
| tlm common | 1 | 40 | 9 | 1 |
| tlm LT | 5 | 28 | 10 | 0 |
| tlm AT | 6 | 35 | 12 | 0 |
| topdown | - | - | 6 | - |
| **Total** | **25** | **226** | **121** | **9** |

## Documentation Type Description

| Type | Description | Example |
| --- | --- | --- |
| `_index.md` | Subsystem overview page with architecture diagram and file list | `code/sysc/pipe/_index.md` |
| `*.md` (per-file) | Detailed analysis corresponding to a source file | `code/sysc/pipe/stage1.md` |
| `spec.md` | Hardware IP specification (with software analogies) | `code/sysc/pipe/spec.md` |
| topdown `*.md` | Cross-example conceptual documentation | `topdown/tlm-explained.md` |

## Software Analogy Quick Reference

| SystemC / Hardware Concept | Software Analogy |
| --- | --- |
| FIFO | Python queue.Queue |
| Pipeline | Unix pipe / ETL chain |
| FIR Filter | Sliding window weighted average |
| FFT | Spectrum analyzer / music equalizer |
| Packet Switch | Network router / RabbitMQ |
| Bus Arbitration | Shared resource + Mutex / thread scheduler |
| RISC CPU | Instruction interpreter + cache system |
| TLM LT | Synchronous HTTP (fetch + await) |
| TLM AT | Asynchronous HTTP (callback/asyncio.Future) |
| TLM DMI | mmap / kernel bypass |
| TLM Extension | Custom HTTP Header |
| sc_module | class / component |
| sc_port | dependency injection |
| sc_signal | Observable / reactive variable |
| SC_THREAD | coroutine / Python coroutine (asyncio) |
| SC_METHOD | event callback |
| sc_event | condition variable / asyncio.Future |
| Delta cycle | microtask queue |
