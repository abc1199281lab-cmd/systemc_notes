# SystemC Object Analysis

> **File**: `ref/systemc/src/sysc/kernel/sc_object.cpp`

## 1. Overview
`sc_object` is the abstract base class for all simulation objects (channels, ports, modules, etc.) in SystemC. It provides:
- **Identification**: Name, Kind.
- **Hierarchy**: Parent-child relationships (via `sc_object_host` and `sc_object_manager`).
- **Attributes**: A mechanism to attach arbitrary metadata (`sc_attr_base`).

## 2. Core Concepts

### 2.1 Object Initialization (`sc_object_init`)
When an `sc_object` is created:
1.  **Simulation Context**: It grabs the current `sc_simcontext`.
2.  **Parenting**: It finds its parent (the `active_object()`) and adds itself to the parent's child list.
3.  **Registration**: It inserts itself into the `sc_object_manager`.

### 2.2 Hierarchy Management
The object hierarchy is maintained dynamically during construction.
- `m_parent`: Pointer to the parent object.
- `get_child_objects()`: (Inherited/Managed via `sc_object_host` usually).

### 2.3 Attributes
SystemC objects support a dynamic attribute system.
- `add_attribute()`, `get_attribute()`: Allows users to attach custom data keys/values to any object, useful for tool-specific backends or tracing.

## 3. Hardware / RTL Mapping
`sc_object` itself typically maps to **Component Instances** or **Signal Names** in a netlist.
- The `name()` (path) of an `sc_object` corresponds to the hierarchical path in a Verilog/VHDL netlist (e.g., `top.cpu.alu.adder`).

## 4. Key Takeaways
1.  **Construction Order Matters**: SystemC builds hierarchy *during execution* of constructors. The "active" object (the one currently running its constructor) becomes the parent of any new objects.
2.  **Unique Names**: The kernel enforces uniqueness. If you try to create two objects with the same name in the same scope, the `sc_object_manager` (via `sc_name_gen`) usually generates a unique suffix.
3.  **Detach**: Objects can be "detached" (removed from hierarchy), but this is rare and dangerous during simulation.
