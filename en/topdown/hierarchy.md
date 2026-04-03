# Module Hierarchy

## Everyday Analogy: Building with LEGO Bricks

The module hierarchy in SystemC is like building a house with LEGO bricks:

- **sc_object** = The base of every LEGO brick -- whether it is a wall, window, or door, it is still a LEGO brick
- **sc_module** = An assembled sub-structure -- for example, "the second-floor bathroom" is a module
- **Port** = A connector on a brick -- lets two sub-structures snap together
- **Export** = A slot on a brick -- accepts connectors from others
- **Channel** = A pipe connecting two connectors -- data flows through here
- **Object tree** = The parts list in the assembly manual -- every part has a name and belongs somewhere

Just as you can pull out the "bathroom module" and swap it with a different design,
SystemC's modular design lets you replace and reuse components.

---

## sc_object: The Root of Everything

`sc_object` is the base class for all named objects in SystemC.
All modules, ports, channels, and processes inherit from it.

```mermaid
classDiagram
    class sc_object {
        -m_name : string
        -m_parent : sc_object*
        -m_child_objects : vector
        +name() const char*
        +basename() const char*
        +get_parent_object() sc_object*
        +get_child_objects() vector
    }

    class sc_module {
        +SC_HAS_PROCESS()
        +SC_METHOD()
        +SC_THREAD()
        +SC_CTHREAD()
    }

    class sc_port_base {
        +bind()
        +operator()
    }

    class sc_export_base {
        +bind()
    }

    class sc_prim_channel {
        +request_update()
        +update()
    }

    class sc_process_b {
        +is_runnable()
        +trigger_static()
    }

    sc_object <|-- sc_module
    sc_object <|-- sc_port_base
    sc_object <|-- sc_export_base
    sc_object <|-- sc_prim_channel
    sc_object <|-- sc_process_b
```

### Hierarchical Naming

Every `sc_object` has a **hierarchical name** that reflects its position in the object tree:

```
top                        # Top-level module
top.cpu                    # cpu sub-module inside top
top.cpu.alu                # alu sub-module inside cpu
top.cpu.alu.port_a         # A port of alu
top.cpu.alu.method_p_0     # A process of alu
```

```mermaid
graph TD
    TOP["top<br/>(sc_module)"]
    CPU["top.cpu<br/>(sc_module)"]
    MEM["top.memory<br/>(sc_module)"]
    ALU["top.cpu.alu<br/>(sc_module)"]
    REG["top.cpu.regs<br/>(sc_module)"]
    PA["top.cpu.alu.port_a<br/>(sc_in)"]
    PB["top.cpu.alu.port_b<br/>(sc_in)"]
    POUT["top.cpu.alu.port_out<br/>(sc_out)"]
    PROC["top.cpu.alu.compute<br/>(SC_METHOD)"]

    TOP --> CPU
    TOP --> MEM
    CPU --> ALU
    CPU --> REG
    ALU --> PA
    ALU --> PB
    ALU --> POUT
    ALU --> PROC

    style TOP fill:#e3f2fd
    style CPU fill:#e3f2fd
    style MEM fill:#e3f2fd
    style ALU fill:#e3f2fd
    style REG fill:#e3f2fd
    style PA fill:#c8e6c9
    style PB fill:#c8e6c9
    style POUT fill:#fff3e0
    style PROC fill:#fce4ec
```

---

## sc_module: The Basic Unit of Design

`sc_module` is the fundamental way to define hardware components in SystemC.
Each module can contain:

1. **Sub-modules** -- smaller components
2. **Ports** -- external interfaces
3. **Processes** -- behavioral descriptions (SC_METHOD, SC_THREAD, SC_CTHREAD)
4. **Internal signals** -- wiring between sub-modules
5. **Internal variables** -- the module's private state

```cpp
SC_MODULE(ALU) {
    // Ports
    sc_in<int>  a, b;
    sc_in<int>  op;
    sc_out<int> result;

    // Process
    void compute() {
        if (op.read() == 0)
            result.write(a.read() + b.read());
        else
            result.write(a.read() - b.read());
    }

    SC_CTOR(ALU) {
        SC_METHOD(compute);
        sensitive << a << b << op;
    }
};
```

### Module Construction Flow

```mermaid
sequenceDiagram
    participant Main as sc_main
    participant MN as sc_module_name
    participant MOD as MyModule
    participant CTX as sc_simcontext

    Main->>MN: Create sc_module_name("mod")
    Note over MN: Push name onto stack
    Main->>MOD: new MyModule("mod")
    MOD->>CTX: Register in object tree
    MOD->>MOD: Create sub-modules
    MOD->>MOD: Create ports/exports
    MOD->>MOD: Register SC_METHOD/SC_THREAD
    MOD->>MN: Destroy sc_module_name
    Note over MN: Pop name from stack
```

---

## Port, Export, and Channel

These three form the communication triad between SystemC modules:

```mermaid
flowchart LR
    subgraph "Module A"
        PROC_A["Process A"]
        PORT["sc_out<br/>(Port)"]
    end

    subgraph "Module B"
        EXPORT["sc_export<br/>(Export)"]
        PROC_B["Process B"]
    end

    SIGNAL["sc_signal<br/>(Channel)"]

    PROC_A -->|write| PORT
    PORT -->|bind to| SIGNAL
    SIGNAL -->|bind to| EXPORT
    EXPORT -->|read| PROC_B

    style PORT fill:#c8e6c9
    style EXPORT fill:#fff3e0
    style SIGNAL fill:#e3f2fd
```

### Port -- The "Plug" of a Module

A port defines what kind of external connection a module needs:

| Port Type | Direction | Analogy |
|-----------|-----------|---------|
| `sc_in<T>` | Input | The data-in wire of a USB cable |
| `sc_out<T>` | Output | The data-out wire of a USB cable |
| `sc_inout<T>` | Bidirectional | A bidirectional I2C data line |

### Export -- The "Socket" of a Module

An export lets a module directly provide an implementation of an interface,
without going through an intermediate channel.

```mermaid
flowchart LR
    subgraph "Module A"
        PA["sc_port<IF>"]
    end

    subgraph "Module B"
        EB["sc_export<IF>"]
        IMPL["Interface implementation"]
        EB -.-> IMPL
    end

    PA -->|direct bind| EB

    style PA fill:#c8e6c9
    style EB fill:#fff3e0
```

### Channel -- The Communication Pipe

A channel is the concrete component that implements an interface, responsible for data transfer and synchronization.

---

## Binding

Binding is the process of connecting ports, exports, and channels together during the elaboration phase.

```mermaid
flowchart TD
    subgraph "Valid Binding Patterns"
        P1["Port → Signal"]
        P2["Port → Export"]
        P3["Port → Port (parent to child module)"]
        P4["Export → Export (child to parent module)"]
    end

    subgraph "Syntax"
        S1["module.port(signal)"]
        S2["module.port(other.export)"]
        S3["child.port(parent.port)"]
        S4["parent.export(child.export)"]
    end

    P1 --- S1
    P2 --- S2
    P3 --- S3
    P4 --- S4
```

### Hierarchical Binding

In multi-level module hierarchies, a port can "cross" levels:

```mermaid
graph TD
    subgraph "Top"
        subgraph "Sub A"
            PA["port_a"]
        end
        SIG["signal_x"]
        subgraph "Sub B"
            PB["port_b"]
        end
    end

    PA -->|bind| SIG
    PB -->|bind| SIG

    style PA fill:#c8e6c9
    style PB fill:#c8e6c9
    style SIG fill:#e3f2fd
```

---

## What Happens During the Elaboration Phase?

```mermaid
flowchart TD
    A[Start Elaboration] --> B["Create all module instances<br/>(outside-in, recursive construction)"]
    B --> C["Create all Ports and Exports"]
    C --> D["Bind Port ↔ Channel ↔ Export"]
    D --> E["Register all Processes<br/>(SC_METHOD, SC_THREAD)"]
    E --> F["Call before_end_of_elaboration()"]
    F --> G["Call end_of_elaboration()"]
    G --> H["Check all Ports are bound"]
    H --> I{Any unbound<br/>Ports?}
    I -->|Yes| ERR[Report error and abort]
    I -->|No| J[Elaboration complete<br/>Enter Simulation]

    style A fill:#c8e6c9
    style J fill:#c8e6c9
    style ERR fill:#ffcdd2
```

### The Clever Design of sc_module_name

`sc_module_name` uses the pairing of constructor and destructor
to automatically manage the module name stack:

```mermaid
sequenceDiagram
    participant Stack as Name Stack

    Note over Stack: Stack is empty

    Note over Stack: MyTop top("top")
    Stack->>Stack: push("top")
    Note over Stack: ["top"]

    Note over Stack: MyCPU cpu("cpu") (inside top's constructor)
    Stack->>Stack: push("cpu")
    Note over Stack: ["top", "cpu"]

    Note over Stack: cpu construction complete
    Stack->>Stack: pop()
    Note over Stack: ["top"]

    Note over Stack: top construction complete
    Stack->>Stack: pop()
    Note over Stack: []
```

---

## How This Maps to Hardware Blocks

```mermaid
graph TD
    subgraph "SystemC Module Hierarchy"
        SC_TOP["sc_module: SoC"]
        SC_CPU["sc_module: CPU"]
        SC_MEM["sc_module: Memory"]
        SC_BUS["sc_module: Bus"]
        SC_ALU["sc_module: ALU"]
        SC_CTRL["sc_module: Controller"]

        SC_TOP --> SC_CPU
        SC_TOP --> SC_MEM
        SC_TOP --> SC_BUS
        SC_CPU --> SC_ALU
        SC_CPU --> SC_CTRL
    end

    subgraph "Corresponding Hardware Blocks"
        HW_SOC["Chip (SoC)"]
        HW_CPU["Processor Core"]
        HW_MEM["Memory Controller"]
        HW_BUS["System Bus"]
        HW_ALU["Arithmetic Logic Unit"]
        HW_CTRL["Control Unit"]

        HW_SOC --> HW_CPU
        HW_SOC --> HW_MEM
        HW_SOC --> HW_BUS
        HW_CPU --> HW_ALU
        HW_CPU --> HW_CTRL
    end

    SC_TOP -.->|maps to| HW_SOC
    SC_CPU -.->|maps to| HW_CPU
    SC_MEM -.->|maps to| HW_MEM

    style SC_TOP fill:#e3f2fd
    style SC_CPU fill:#e3f2fd
    style SC_MEM fill:#e3f2fd
    style HW_SOC fill:#fff3e0
    style HW_CPU fill:#fff3e0
    style HW_MEM fill:#fff3e0
```

In hardware design:
- **Module** corresponds to a hardware functional block (IP Block)
- **Port** corresponds to chip pins or block inputs/outputs
- **Signal** corresponds to a physical wire
- **Object tree** corresponds to a hierarchical block diagram of the hardware

---

## sc_object_manager and sc_module_registry

These two classes manage all objects and modules behind the scenes:

```mermaid
classDiagram
    class sc_object_manager {
        -m_object_table : name-to-object lookup table
        +find_object(name) sc_object*
        +insert_object(name, obj)
        +remove_object(name)
        +hierarchy_push(obj)
        +hierarchy_pop()
        +hierarchy_curr() sc_object*
    }

    class sc_module_registry {
        -m_module_vec : vector~sc_module~
        +insert(module)
        +remove(module)
        +all() vector
    }

    class sc_simcontext {
    }

    sc_simcontext --> sc_object_manager : owns
    sc_simcontext --> sc_module_registry : owns
```

---

## Related Modules

| Concept | File | Relationship |
|---------|------|--------------|
| Simulation Engine | [simulation-engine.md](simulation-engine.md) | Elaboration is the first phase of the simulation engine lifecycle |
| Communication | [communication.md](communication.md) | Detailed explanation of the Port-Channel-Export pattern |
| Events | [events.md](events.md) | Processes are driven by events |
| Scheduling | [scheduling.md](scheduling.md) | Process scheduling behavior |

### Corresponding Source Code Files

| Source Code Concept | Code File |
|---------------------|-----------|
| sc_object | [doc_v2/code/sysc/kernel/sc_object.md](../code/sysc/kernel/sc_object.md) |
| sc_module | [doc_v2/code/sysc/kernel/sc_module.md](../code/sysc/kernel/sc_module.md) |
| sc_module_name | [doc_v2/code/sysc/kernel/sc_module_name.md](../code/sysc/kernel/sc_module_name.md) |
| sc_module_registry | [doc_v2/code/sysc/kernel/sc_module_registry.md](../code/sysc/kernel/sc_module_registry.md) |
| sc_object_manager | [doc_v2/code/sysc/kernel/sc_object_manager.md](../code/sysc/kernel/sc_object_manager.md) |
| sc_port | [doc_v2/code/sysc/communication/sc_port.md](../code/sysc/communication/sc_port.md) |
| sc_export | [doc_v2/code/sysc/communication/sc_export.md](../code/sysc/communication/sc_export.md) |

---

## Learning Tips

1. **sc_object is like Python's `object` base class** -- all named SystemC objects inherit from it
2. **Module = container** -- it does not "do things" itself; the processes inside it do the work
3. **Port defines "what is needed," Channel provides "how it is done"** -- this is the classic separation of interface and implementation
4. **Hierarchical name is the object's address** -- `top.cpu.alu` uniquely identifies any object
5. **Once elaboration is complete, the structure is fixed** -- you cannot add or remove modules during simulation
6. **Drawing the object tree is the first step to understanding a design** -- when you get a SystemC design, start by drawing the module hierarchy diagram
