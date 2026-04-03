# Hardware Specification -- Packet Switch

> This article is written for software engineers, explaining the hardware background of a packet switch.

## What Is a Packet Switch?

A packet switch is a hardware device that forwards data packets from one input port to one or more output ports.

**Everyday Examples**:
- The **Ethernet switch** under your desk is a packet switch
- The **WiFi router** in your home also contains a packet switch internally
- **Top-of-Rack switches** in data centers handle all server traffic within a rack

**Software Analogy**:

| Hardware Concept | Software Analogy |
|-----------------|-----------------|
| Packet switch | Load balancer (e.g., Nginx, HAProxy) |
| Input port | Incoming TCP connection |
| Output port | Backend server |
| Routing table | Nginx `upstream` configuration |
| Multicast | Pub/sub fanout (e.g., RabbitMQ exchange) |
| Buffer overflow (drop) | HTTP 503 Service Unavailable |

## Why Do We Need Hardware Switches?

Software routers (e.g., Linux's iptables) can do the same thing, but hardware switches have an overwhelming performance advantage:

| Comparison | Software Routing | Hardware Switch |
|-----------|-----------------|-----------------|
| Latency | Microsecond to millisecond range | Nanosecond range (hundreds of times faster) |
| Throughput | Gbps range | Tens of Tbps range |
| CPU usage | High (software runs for every packet) | Zero (dedicated circuits handle it) |
| Power consumption | High | Low (fixed power) |
| Flexibility | High (can change the program anytime) | Low (circuits are fixed) |

**Key Difference**: Hardware switches can achieve **wire-speed routing** -- the processing speed matches the incoming packet rate, creating no bottleneck. Software cannot achieve this because every packet requires CPU intervention.

## Switch Architecture Comparison

### Crossbar

```mermaid
graph TB
    subgraph "Crossbar Switch"
        I0["Input 0"] --> X00["X"] --> O0["Output 0"]
        I0 --> X01["X"] --> O1["Output 1"]
        I0 --> X02["X"] --> O2["Output 2"]
        I1["Input 1"] --> X10["X"] --> O0
        I1 --> X11["X"] --> O1
        I1 --> X12["X"] --> O2
        I2["Input 2"] --> X20["X"] --> O0
        I2 --> X21["X"] --> O1
        I2 --> X22["X"] --> O2
    end
```

- **Principle**: Every input has a direct path to every output. Like an NxN switch matrix.
- **Pros**: Non-blocking (any input to any output without interference), low latency
- **Cons**: Hardware cost is O(N^2), cost explodes as port count increases
- **Software Analogy**: Full mesh topology, with a direct connection between every pair of services

### Shared Bus

```mermaid
graph LR
    I0["Input 0"] --> BUS["Shared Bus"]
    I1["Input 1"] --> BUS
    I2["Input 2"] --> BUS
    BUS --> O0["Output 0"]
    BUS --> O1["Output 1"]
    BUS --> O2["Output 2"]
```

- **Principle**: All ports share a single data channel; only one port can transmit at a time
- **Pros**: Low hardware cost (O(N))
- **Cons**: Bandwidth bottleneck (all traffic competes for one path), requires arbitration
- **Software Analogy**: A single message queue where all producers write to the same queue

### Ring -- The Architecture Used in This Example

```mermaid
graph LR
    R0["Register 0"] --> R1["Register 1"]
    R1 --> R2["Register 2"]
    R2 --> R3["Register 3"]
    R3 --> R0

    I0["Input 0"] -.-> R0
    I1["Input 1"] -.-> R1
    I2["Input 2"] -.-> R2
    I3["Input 3"] -.-> R3

    R0 -.-> O0["Output 0"]
    R1 -.-> O1["Output 1"]
    R2 -.-> O2["Output 2"]
    R3 -.-> O3["Output 3"]
```

- **Principle**: Packets rotate around the ring, checking at each output port whether they match. If they match, a copy is sent out.
- **Pros**: Moderate hardware cost (O(N)), naturally supports multicast (a packet can reach all destinations in one full rotation)
- **Cons**: Higher latency (up to N cycles to reach the destination)
- **Software Analogy**: Token ring network, or Kafka's partition rebalance -- messages are passed around among a group of consumers in turn

### Architecture Comparison Summary

| Architecture | Hardware Cost | Latency | Bandwidth | Multicast | Software Analogy |
|-------------|--------------|---------|-----------|-----------|-----------------|
| Crossbar | O(N^2) | 1 cycle | Highest | Requires extra logic | Full mesh |
| Shared Bus | O(N) | Variable | Lowest | Easy (broadcast) | Single queue |
| Ring | O(N) | 1~N cycles | Medium | Naturally supported | Token ring |

## The Helix Packet Switch in This Example

This example implements a **Helix packet switch**, a variant of the ring architecture:

```mermaid
flowchart TD
    subgraph "Helix Switch Operation Flow"
        A["Packet enters input FIFO"] --> B["Load into empty shift register"]
        B --> C["Ring rotation (rotates once per control cycle)"]
        C --> D{"Register position<br/>matches destination?"}
        D -- "Yes" --> E["Copy packet to output FIFO"]
        E --> F{"All destinations<br/>delivered?"}
        F -- "Yes" --> G["Release register"]
        F -- "No" --> C
        D -- "No" --> C
    end
```

**The Cleverness of Helix**:

1. **Self-routing**: No centralized routing table needed. Packets carry their own destination information, and the register position determines which output it can write to.
2. **Pipelined**: Multiple packets can flow through the ring simultaneously without waiting for the previous one to finish.
3. **Non-blocking multicast**: Packets stay in the ring rotating until all destinations are served, without blocking other packets.

## Real-World Applications

### Ethernet Switch

The Ethernet switch under your desk is the most common packet switch. It typically has 4, 8, 24, or 48 ports. Enterprise-grade switches use crossbar or more complex multi-stage architectures.

### Network-on-Chip (NoC)

Modern multi-core processors (e.g., AMD EPYC, Intel Xeon) have dozens of CPU cores internally, and they communicate through a NoC -- a packet-switching network inside the chip. Common NoC topologies include ring (Intel's early multi-core designs) and mesh (AMD's chiplet architecture).

```mermaid
graph TB
    subgraph "NoC Mesh (4x4)"
        C00["Core"] --- C01["Core"] --- C02["Core"] --- C03["Core"]
        C10["Core"] --- C11["Core"] --- C12["Core"] --- C13["Core"]
        C20["Core"] --- C21["Core"] --- C22["Core"] --- C23["Core"]
        C30["Core"] --- C31["Core"] --- C32["Core"] --- C33["Core"]

        C00 --- C10
        C01 --- C11
        C02 --- C12
        C03 --- C13
        C10 --- C20
        C11 --- C21
        C12 --- C22
        C13 --- C23
        C20 --- C30
        C21 --- C31
        C22 --- C32
        C23 --- C33
    end
```

### PCIe Switch

PCIe switches that connect high-speed devices like GPUs, NVMe SSDs, and network cards are also a type of packet switch. They allow multiple devices to share PCIe lanes.

### Similar Concepts in Software

If you are writing distributed systems, you may already be familiar with these concepts:

| Hardware | Software |
|----------|----------|
| Packet switch | Service mesh sidecar (Envoy) |
| Input FIFO | Request queue |
| Output FIFO | Response buffer |
| Multicast | gRPC server streaming / Kafka topic fanout |
| Packet drop (FIFO full) | Circuit breaker / backpressure / HTTP 429 |
| Wire-speed routing | Kernel bypass networking (DPDK, io_uring) |

## Why Simulate with SystemC?

The typical flow for designing a packet switch is:

1. **Algorithm exploration** (SystemC / C++ model) -- the stage this example is at
2. **RTL design** (Verilog / VHDL)
3. **Verification** (using the SystemC model as a golden reference)
4. **Synthesis** (converting to actual logic gate circuits)
5. **Manufacturing** (sent to the foundry)

SystemC allows hardware designers to quickly verify switch behavior in C++: packet drop rates under different buffer sizes, different routing strategies, and different traffic patterns. This is much faster than writing RTL directly and is also easier to debug.
