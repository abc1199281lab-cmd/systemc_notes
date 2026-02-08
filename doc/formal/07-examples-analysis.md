# Phase 7: Example Programs Analysis

> Date: 2026-02-08  
> Objective: Analyze SystemC and TLM examples to understand practical patterns

---

## 7.1 SystemC Basic Examples

### 7.1.1 Simple FIFO - Producer/Consumer Pattern

**File**: `examples/sysc/simple_fifo/simple_fifo.cpp`

This is the classic "Hello World" of SystemC - demonstrating basic channel communication.

#### Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│  Producer   │──────▶│    FIFO     │◀─────│  Consumer   │
│  (THREAD)   │ write │  (CHANNEL)  │ read │  (THREAD)   │
└─────────────┘      └─────────────┘      └─────────────┘
        │                    │                    │
        └──────── sc_event ──┴──── sc_event ─────┘
        (write_event)      (read_event)
```

#### Key Code Patterns

```cpp
// 1. Interface Definition (separates read/write)
class write_if : virtual public sc_interface {
public:
    virtual void write(char) = 0;
    virtual void reset() = 0;
};

class read_if : virtual public sc_interface {
public:
    virtual void read(char &) = 0;
    virtual int num_available() = 0;
};

// 2. Channel Implementation (the actual FIFO)
class fifo : public sc_channel, public write_if, public read_if {
private:
    enum e { max = 10 };
    char data[max];
    int num_elements, first;
    sc_event write_event, read_event;  // For blocking communication

public:
    void write(char c) {
        if (num_elements == max)
            wait(read_event);  // Block if FIFO full
        
        data[(first + num_elements) % max] = c;
        ++num_elements;
        write_event.notify();  // Wake up reader
    }
    
    void read(char &c) {
        if (num_elements == 0)
            wait(write_event);  // Block if FIFO empty
        
        c = data[first];
        --num_elements;
        first = (first + 1) % max;
        read_event.notify();  // Wake up writer
    }
};

// 3. Module with Port
class producer : public sc_module {
public:
    sc_port<write_if> out;  // Uses interface, not concrete channel
    
    void main() {
        const char *str = "Visit www.accellera.org...";
        while (*str)
            out->write(*str++);  // Write to FIFO
    }
};
```

#### Design Patterns Demonstrated

1. **Interface-Based Design**: Modules use interfaces (`write_if`, `read_if`), not concrete types
2. **Event-Based Synchronization**: `wait()` and `notify()` for blocking communication
3. **Circular Buffer**: `%(max)` for efficient FIFO implementation
4. **SC_THREAD**: Each module runs in its own thread

#### RTL Mapping

| SystemC Pattern | RTL Equivalent |
|-----------------|----------------|
| `sc_event` | Clock edge or handshaking signal |
| `wait(event)` | Wait for posedge clk or ready/valid handshake |
| `notify()` | Signal assertion |
| FIFO channel | FIFO buffer with full/empty flags |
| Producer/Consumer | Master/Slave with flow control |

---

### 7.1.2 RISC CPU Example

**File**: `examples/sysc/risc_cpu/main.cpp`

A complete CPU model with pipeline stages - demonstrates complex SystemC system modeling.

#### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         RISC CPU Pipeline                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐     │
│  │  Fetch  │───▶│ Decode  │───▶│ Execute │───▶│  DCACHE │     │
│  │  (IFU)  │    │  (IDU)  │    │  (IEU)  │    │         │     │
│  └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘     │
│       │              │              │              │            │
│       ▼              ▼              ▼              ▼            │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐     │
│  │  ICACHE │    │ Branch  │    │  Float  │    │  Memory │     │
│  │         │    │  Pred   │    │  (FPU)  │    │         │     │
│  └─────────┘    │  (BPU)  │    │  (MMX)  │    └─────────┘     │
│  ┌─────────┐    └─────────┘    └─────────┘                   │
│  │  BIOS   │                                                  │
│  │         │                                                  │
│  └─────────┘                                                  │
│  ┌─────────┐                                                  │
│  │ Paging  │                                                  │
│  │         │                                                  │
│  └─────────┘                                                  │
│  ┌─────────┐                                                  │
│  │  PIC    │  (Programmable Interrupt Controller)            │
│  │         │                                                  │
│  └─────────┘                                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Key Components

1. **Fetch (IFU)**: Fetches instructions from ICACHE/BIOS
2. **Decode (IDU)**: Decodes instructions, handles forwarding
3. **Execute (IEU)**: ALU operations
4. **Floating Point (FPU)**: Floating point unit
5. **MMX Unit (MMXU)**: SIMD operations
6. **ICACHE/DCACHE**: Instruction and data caches
7. **Paging**: Memory management unit
8. **PIC**: Interrupt controller

#### Pipeline Signals

```cpp
// Between Fetch and Decode
sc_signal<unsigned>    instruction("INSTRUCTION");
sc_signal<bool>        instruction_valid("INSTRUCTION_VALID");
sc_signal<unsigned>    program_counter("PROGRAM_COUNTER");

// Between Decode and Execute
sc_signal<int>         alu_op("ALU_OP");
sc_signal<signed int>  src_A("SRC_A");
sc_signal<signed int>  src_B("SRC_B");
sc_signal<bool>        decode_valid("DECODE_VALID");

// Memory interface
sc_signal<bool>        mem_access("MEM_ACCESS");
sc_signal<unsigned>    mem_address("MEM_ADDRESS");
sc_signal<bool>        mem_write("MEM_WRITE");
```

#### Design Patterns

1. **Pipeline Stages**: Each stage is a separate SC_MODULE
2. **Signal Connectivity**: `sc_signal` connects pipeline stages
3. **Forwarding Logic**: Decode stage checks for data hazards
4. **Stall Handling**: Pipeline stalls via `stall_fetch` signal

---

## 7.2 TLM-2.0 Examples

### 7.2.1 Loosely-Timed (LT) Example

**Directory**: `examples/tlm/lt/`

LT modeling is the highest performance TLM style - used for virtual platforms and software development.

#### Key Characteristics

```cpp
// Typical LT initiator pattern
class lt_initiator : public sc_module {
    tlm_utils::simple_initiator_socket<lt_initiator> socket;
    tlm_utils::tlm_quantumkeeper m_qk;  // For temporal decoupling
    
    void run() {
        while (true) {
            // Create transaction
            tlm_generic_payload trans;
            trans.set_address(addr);
            trans.set_command(TLM_WRITE_COMMAND);
            trans.set_data_ptr(data);
            trans.set_data_length(size);
            
            // Set local time offset
            sc_time delay = m_qk.get_local_time();
            
            // Blocking transport - returns when complete
            socket->b_transport(trans, delay);
            
            // Update local time
            m_qk.inc(delay);
            
            // Sync if needed (based on global quantum)
            if (m_qk.need_sync()) {
                m_qk.sync();
            }
        }
    }
};
```

#### Temporal Decoupling Concept

```
┌───────────────────────────────────────────────────────────┐
│               Temporal Decoupling Explained               │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  SystemC Time  0    10    20    30    40    50    60    │
│                 │     │     │     │     │     │     │   │
│                 ▼     ▼     ▼     ▼     ▼     ▼     ▼   │
│  Global Quantum ├───────────────┤ (e.g., 40ns)           │
│                                                           │
│  Initiator A    [==========]wait[==========]wait...      │
│                 local=10         local=20                  │
│                                                           │
│  Initiator B    [====================]wait[====]...    │
│                 local=30                                          │
│                                                           │
│  All initiators sync at quantum boundaries                 │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### 7.2.2 Approximately-Timed (AT) Example

**Directory**: `examples/tlm/at_1_phase/`, `at_2_phase/`, `at_4_phase/`

AT modeling provides more timing accuracy for performance analysis.

#### 2-Phase Protocol

```cpp
// Initiator side
void initiator_thread() {
    tlm_generic_payload trans;
    tlm_phase phase = BEGIN_REQ;
    sc_time delay = SC_ZERO_TIME;
    
    // Send request
    tlm_sync_enum status = socket->nb_transport_fw(trans, phase, delay);
    
    // Wait for response phase
    if (status == TLM_UPDATED) {
        wait(phase_event);  // Wait for END_RESP
    }
}

// Target callback
virtual tlm_sync_enum nb_transport_fw(tlm_generic_payload& trans,
                                      tlm_phase& phase,
                                      sc_time& delay) {
    if (phase == BEGIN_REQ) {
        // Process request
        phase = END_RESP;  // Indicate response ready
        return TLM_UPDATED; // Tell initiator to wait
    }
    return TLM_COMPLETED;
}
```

#### Phase Flow

```
┌────────────────────────────────────────────────────────────┐
│                    2-Phase Protocol                          │
├────────────────────────────────────────────────────────────┤
│                                                              │
│  Initiator                  Target                           │
│     │                         │                              │
│     │  BEGIN_REQ              │                              │
│     │─────────────────────────▶│                              │
│     │                         │                              │
│     │         TLM_UPDATED     │                              │
│     │◀─────────────────────────│                              │
│     │                         │                              │
│     │       (time passes)     │                              │
│     │                         │                              │
│     │  END_RESP               │                              │
│     │◀─────────────────────────│                              │
│     │                         │                              │
│     │         TLM_COMPLETED   │                              │
│     │─────────────────────────▶│                              │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

### 7.2.3 TLM with DMI Example

**Directory**: `examples/tlm/lt_dmi/`

Demonstrates Direct Memory Interface for high-speed memory access.

#### DMI Pattern

```cpp
class dmi_initiator : public sc_module {
    tlm_dmi m_dmi_info;  // Cached DMI info
    bool m_dmi_valid;
    
    void run() {
        while (true) {
            if (!m_dmi_valid) {
                // Get DMI pointer first
                tlm_generic_payload trans;
                trans.set_address(addr);
                m_dmi_valid = socket->get_direct_mem_ptr(trans, m_dmi_info);
            }
            
            if (m_dmi_valid && 
                addr >= m_dmi_info.get_start_address() &&
                addr <= m_dmi_info.get_end_address()) {
                // Direct memory access - very fast!
                unsigned char* ptr = m_dmi_info.get_dmi_ptr();
                memcpy(ptr + (addr - m_dmi_info.get_start_address()), 
                       data, size);
            } else {
                // Fall back to regular transaction
                socket->b_transport(trans, delay);
            }
        }
    }
    
    // Called by target when DMI region changes
    virtual void invalidate_direct_mem_ptr(sc_dt::uint64 start, sc_dt::uint64 end) {
        if (m_dmi_valid) {
            // Check if our cached region is affected
            if (regions_overlap(start, end, m_dmi_info)) {
                m_dmi_valid = false;  // Invalidate cache
            }
        }
    }
};
```

---

## 7.3 Example Analysis Summary

### 7.3.1 Common Patterns

| Pattern | Example File | Use Case |
|---------|-------------|----------|
| FIFO Channel | `simple_fifo.cpp` | Basic communication |
| Pipeline CPU | `risc_cpu/` | Complex RTL modeling |
| LT Initiator | `lt/src/` | Software development |
| AT Initiator | `at_1_phase/` | Performance analysis |
| DMI Access | `lt_dmi/` | High-speed memory |

### 7.3.2 Module Types Comparison

| Aspect | SystemC RTL | TLM-2.0 LT | TLM-2.0 AT |
|--------|-------------|-----------|-----------|
| Timing | Cycle-accurate | Loosely-timed | Approximately-timed |
| Speed | Slow | Fastest | Fast |
| Accuracy | Exact | Approximate | Phase-accurate |
| Use Case | Implementation | Software dev | Architecture |
| Process | SC_METHOD/THREAD | SC_THREAD | SC_THREAD |
| Communication | sc_signal/sc_port | b_transport | nb_transport |

---

## Summary

SystemC examples demonstrate:

1. **Interface-Based Design**: Decouple modules through interfaces
2. **Event Synchronization**: Use events for blocking communication
3. **Temporal Decoupling**: LT modeling for speed
4. **Phase-Based Protocols**: AT modeling for accuracy
5. **DMI Optimization**: Direct access for memory-intensive operations

These patterns form the foundation for building complex system-level models.
