# Phase 6: TLM (Transaction Level Modeling) Analysis

> Date: 2026-02-08  
> Objective: Deep dive into TLM-2.0 and TLM-1.0 design principles

---

## 6.1 TLM Basic Concepts

### 6.1.1 What is TLM?

**TLM (Transaction Level Modeling)** is a SystemC methodology for high-level system modeling. Unlike RTL modeling, TLM doesn't track signal changes per clock cycle - it focuses on "transaction" level communication.

#### Analogy
| Modeling Level | Software Analogy | Hardware Equivalent |
|---------------|------------------|---------------------|
| RTL | Assembly (track every cycle) | Verilog/VHDL |
| Cycle-Accurate | Cycle-accurate simulator | Bus functional model |
| TLM-2.0 AT | API library calls | AMBA AXI protocol |
| TLM-2.0 LT | Direct memory access | DMA/Cache |

### 6.1.2 TLM Abstraction Levels

```
┌─────────────────────────────────────────────┐
│         System Level Modeling               │
│       (Virtual Platform / ESL)              │
├─────────────────────────────────────────────┤
│  TLM-2.0 Loosely-Timed (LT)                 │
│  • No timing accuracy                        │
│  • Uses temporal decoupling                  │
│  • For software development                  │
├─────────────────────────────────────────────┤
│  TLM-2.0 Approximately-Timed (AT)           │
│  • Timing considered but not cycle-accurate  │
│  • Uses 2-phase or 4-phase protocols         │
│  • For performance analysis                    │
├─────────────────────────────────────────────┤
│  Cycle-Accurate / RTL                       │
│  • Exact per-cycle timing                    │
│  • SystemC RTL modeling                      │
│  • For implementation verification           │
└─────────────────────────────────────────────┘
```

---

## 6.2 TLM-2.0 Generic Payload

**File**: `src/tlm_core/tlm_2/tlm_generic_payload/tlm_gp.h`

Generic Payload is the standard transaction format in TLM-2.0 - like a "network packet" format that all components understand.

### 6.2.1 Core Attributes

```cpp
class tlm_generic_payload {
    sc_dt::uint64        m_address;          // Memory address
    tlm_command          m_command;          // READ/WRITE/IGNORE
    unsigned char*       m_data;             // Data pointer
    unsigned int         m_length;           // Data length (bytes)
    tlm_response_status  m_response_status;  // Response status
    bool                 m_dmi;              // DMI hint
    unsigned char*       m_byte_enable;      // Byte enable mask
    unsigned int         m_byte_enable_length;
    unsigned int         m_streaming_width;  // Streaming width
};
```

### 6.2.2 Command Types

```cpp
enum tlm_command {
    TLM_READ_COMMAND,    // Read transaction
    TLM_WRITE_COMMAND,   // Write transaction
    TLM_IGNORE_COMMAND   // Ignore (for side-band)
};
```

### 6.2.3 Response Status

```cpp
enum tlm_response_status {
    TLM_OK_RESPONSE = 1,                // Success
    TLM_INCOMPLETE_RESPONSE = 0,        // Incomplete
    TLM_GENERIC_ERROR_RESPONSE = -1,    // Generic error
    TLM_ADDRESS_ERROR_RESPONSE = -2,    // Address error
    TLM_COMMAND_ERROR_RESPONSE = -3,    // Command error
    TLM_BURST_ERROR_RESPONSE = -4,      // Burst error
    TLM_BYTE_ENABLE_ERROR_RESPONSE = -5 // Byte enable error
};
```

### 6.2.4 Extension Mechanism

```cpp
// Custom extension example
class my_extension : public tlm_extension<my_extension> {
public:
    int priority;           // Priority level
    bool cache_coherent;    // Cache coherency required
};

// Usage
tlmm_generic_payload trans;
my_extension* ext = new my_extension;
ext->priority = 7;
trans.set_extension(ext);
```

---

## 6.3 TLM-2.0 Interfaces

**File**: `src/tlm_core/tlm_2/tlm_2_interfaces/tlm_fw_bw_ifs.h`

### 6.3.1 Core Interface Architecture

TLM-2.0 defines "Forward" and "Backward" interfaces forming a **bidirectional communication channel**.

```
┌─────────────────┐         ┌─────────────────┐
│   Initiator     │         │     Target      │
│                 │  ───►   │                 │
│  tlm_initiator  │  fw_if  │   tlm_target    │
│    _socket      │         │    _socket      │
│                 │  ◄───   │                 │
│                 │  bw_if  │                 │
└─────────────────┘         └─────────────────┘
```

### 6.3.2 Forward Interface

```cpp
template <typename TRANS = tlm_generic_payload,
          typename PHASE = tlm_phase>
class tlm_fw_nonblocking_transport_if : public virtual sc_core::sc_interface {
public:
    virtual tlm_sync_enum nb_transport_fw(TRANS& trans,
                                          PHASE& phase,
                                          sc_core::sc_time& t) = 0;
};

template <typename TRANS = tlm_generic_payload>
class tlm_blocking_transport_if : public virtual sc_core::sc_interface {
public:
    virtual void b_transport(TRANS& trans,
                            sc_core::sc_time& t) = 0;
};
```

### 6.3.3 Backward Interface

```cpp
template <typename TRANS = tlm_generic_payload,
          typename PHASE = tlm_phase>
class tlm_bw_nonblocking_transport_if : public virtual sc_core::sc_interface {
public:
    virtual tlm_sync_enum nb_transport_bw(TRANS& trans,
                                          PHASE& phase,
                                          sc_core::sc_time& t) = 0;
};
```

### 6.3.4 Synchronization States

```cpp
enum tlm_sync_enum {
    TLM_ACCEPTED,    // Transaction accepted, continue
    TLM_UPDATED,     // Transaction updated, need wait
    TLM_COMPLETED    // Transaction completed
};
```

### 6.3.5 Phase Mechanism

**File**: `src/tlm_core/tlm_2/tlm_generic_payload/tlm_phase.h`

```cpp
enum tlm_phase_enum {
    UNINITIALIZED_PHASE = 0,  // Uninitialized
    BEGIN_REQ = 1,            // Request begins
    END_REQ,                  // Request ends
    BEGIN_RESP,               // Response begins
    END_RESP                  // Response ends
};
```

**2-Phase Protocol**:
```
Initiator ──BEGIN_REQ──► Target
Initiator ◄──END_RESP── Target
```

**4-Phase Protocol**:
```
Initiator ──BEGIN_REQ──► Target
Initiator ◄──END_REQ──── Target
Initiator ◄──BEGIN_RESP─ Target
Initiator ──END_RESP────► Target
```

---

## 6.4 TLM-2.0 Sockets

**File**: `src/tlm_core/tlm_2/tlm_sockets/tlm_initiator_socket.h`, `tlm_target_socket.h`

### 6.4.1 Socket Design Concept

Socket combines **Port** (sending) and **Export** (receiving) in one entity.

```cpp
// Traditional SystemC (verbose)
sc_port<fw_if> fw_port;
sc_export<bw_if> bw_export;
// Need to connect both separately

// Socket (clean)
tlm_initiator_socket<> socket;
// One socket handles both directions
```

### 6.4.2 Socket Connection

```
┌─────────────────────────────────────────────────────────────┐
│                   Socket Connection Diagram                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Initiator Socket              Target Socket                │
│  ┌─────────────────┐        ┌─────────────────┐          │
│  │  sc_port<fw_if> │───────►│ sc_export<fw_if>│          │
│  │  (send request) │        │ (receive request)│          │
│  └─────────────────┘        └─────────────────┘          │
│         ▲                           │                     │
│         │                           ▼                     │
│  ┌─────────────────┐        ┌─────────────────┐          │
│  │sc_export<bw_if> │◄───────│  sc_port<bw_if> │          │
│  │ (receive resp)  │        │  (send response) │          │
│  └─────────────────┘        └─────────────────┘          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6.5 Direct Memory Interface (DMI)

**File**: `src/tlm_core/tlm_2/tlm_2_interfaces/tlm_dmi.h`

DMI allows Initiator to **directly access Target memory**, skipping interface call overhead. This is key to TLM-2.0's high simulation speed.

### 6.5.1 DMI Access Rights

```cpp
enum dmi_access_e {
    DMI_ACCESS_NONE        = 0x00,  // No access
    DMI_ACCESS_READ        = 0x01,  // Read only
    DMI_ACCESS_WRITE       = 0x02,  // Write only
    DMI_ACCESS_READ_WRITE  = 0x03   // Read/Write
};
```

### 6.5.2 DMI Usage Flow

```
┌──────────────┐                    ┌──────────────┐
│  Initiator   │                    │   Target     │
└──────┬───────┘                    └──────┬───────┘
       │                                   │
       │  1. get_direct_mem_ptr()           │
       │───────────────────────────────────►│
       │                                   │
       │◄────────────────────────────────────│
       │      2. Return tlm_dmi (with ptr)   │
       │                                   │
       │  3. Direct memory access            │
       │  *(dmi_ptr + offset) = data        │
       │                                   │
       │  4. invalidate_direct_mem_ptr()    │
       │◄────────────────────────────────────│
       │      (when memory region changes)   │
```

---

## 6.6 TLM Utils - Convenience Tools

**File**: `src/tlm_utils/README.txt`

### 6.6.1 Simple Socket

```cpp
template< typename MODULE, unsigned int BUSWIDTH = 32 >
class simple_initiator_socket : public tlm::tlm_initiator_socket<...> {
    // Provides default implementations
    // Allows registering callback functions
};
```

### 6.6.2 Quantum Keeper

**File**: `src/tlm_utils/tlm_quantumkeeper.h`

```cpp
class tlm_quantumkeeper {
public:
    static void set_global_quantum(const sc_core::sc_time& t);
    static const sc_core::sc_time& get_global_quantum();
    
    void inc(const sc_core::sc_time& t);     // Increment local time
    void set(const sc_core::sc_time& t);     // Set local time
    bool need_sync() const;                   // Check if sync needed
    void sync();                              // Synchronize to SystemC
    void reset();                             // Reset local time
};
```

---

## 6.7 TLM-1.0 Overview

**File**: `src/tlm_core/tlm_1/README.txt`

TLM-1.0 provides simpler request-response and FIFO-based communication:

```cpp
// Core interfaces in TLM-1.0
tlm_transport_if          // Blocking transport
tlm_blocking_get_if       // Blocking get
tlm_blocking_put_if       // Blocking put
tlm_nonblocking_get_if    // Non-blocking get
tlm_nonblocking_put_if    // Non-blocking put
```

---

## Summary

TLM-2.0 provides:
- **Generic Payload**: Standard transaction format
- **Socket**: Bidirectional connection mechanism
- **LT/AT**: Two timing abstraction levels
- **DMI**: Direct memory access for speed
- **Extensions**: Customizable transaction attributes

**RTL Mapping**:
| TLM-2.0 Concept | RTL Equivalent |
|-----------------|----------------|
| Generic Payload | AXI transaction |
| Socket | AXI master/slave interface |
| Phase | AXI handshake signals |
| DMI | Memory-mapped I/O |
