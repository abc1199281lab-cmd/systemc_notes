# LT + DMI Example Overview

## Software Analogy: Fast Memory Access with mmap

In regular file I/O, every read/write requires a system call (`read()`/`write()`), and the kernel needs to perform permission checks, copy data, etc. But if you use `mmap()`, the OS maps the file directly into your program's memory space, and from then on you can read and write the file as if accessing a regular variable -- no system calls needed, much faster.

TLM's DMI (Direct Memory Interface) is the same concept:

| Regular File I/O | TLM Regular Transaction |
|---|---|
| `read(fd, buf, len)` | `b_transport(payload, delay)` |
| Requires a system call each time | Requires bus routing each time |
| Safe but slow | Accurate but slow |

| mmap | TLM DMI |
|---|---|
| `ptr = mmap(fd)` | `get_direct_mem_ptr()` to obtain a memory pointer |
| `*ptr = data` | Read/write directly via pointer |
| Fast, bypasses the kernel | Fast, bypasses the bus and target |

## Differences from Basic LT

In the basic LT example, every read/write goes through the full `b_transport()` call chain (initiator -> bus -> target -> bus -> initiator). With DMI added:

1. The first transaction still takes the normal path
2. The target tells the initiator: "you can access my memory directly via DMI"
3. Subsequent transactions allow the initiator to read/write the target's memory directly via pointer, bypassing the bus

## System Architecture

```mermaid
graph LR
    subgraph Initiator_1["initiator_top (ID=101)"]
        TG1["traffic_generator"]
        LDI1["lt_dmi_initiator"]
        TG1 -->|request_fifo| LDI1
        LDI1 -->|response_fifo| TG1
    end

    subgraph Initiator_2["initiator_top (ID=102)"]
        TG2["traffic_generator"]
        LDI2["lt_dmi_initiator"]
        TG2 -->|request_fifo| LDI2
        LDI2 -->|response_fifo| TG2
    end

    subgraph Bus["SimpleBusLT<2,2>"]
        Router["Address Router"]
    end

    subgraph Target_1["lt_dmi_target (ID=201)"]
        MEM1["Memory 4KB"]
    end

    subgraph Target_2["lt_dmi_target (ID=202)"]
        MEM2["Memory 4KB"]
    end

    LDI1 -->|"b_transport (normal path)"| Router
    LDI1 -.->|"DMI direct access (fast path)"| MEM1
    LDI1 -.->|"DMI direct access (fast path)"| MEM2
    LDI2 -->|"b_transport (normal path)"| Router
    Router -->|"initiator_socket[0]"| MEM1
    Router -->|"initiator_socket[1]"| MEM2
```

Dashed lines represent the DMI fast path -- bypassing the bus and accessing memory directly.

## Source Files

| File | Description |
|---|---|
| `src/lt_dmi.cpp` | Program entry point `sc_main` |
| `include/lt_dmi_top.h` / `src/lt_dmi_top.cpp` | Top-level module, with component instantiation, connections, and simulation time limit |
| `include/initiator_top.h` / `src/initiator_top.cpp` | Initiator wrapper module, using `lt_dmi_initiator` |

For detailed source code analysis, see [lt-dmi.md](lt-dmi.md).
