# sc_stub.h - Stub Channel, Unbound and Tied-Value Connections

## Overview

This file provides three important utilities for handling scenarios where module ports "don't need actual connections":

1. **`sc_stub<T>`** - A do-nothing fake channel (stub channel)
2. **`sc_unbound`** - Global object for marking "this port is intentionally not connected"
3. **`sc_tie::value(val)`** - Function for binding a port to a fixed value

These utilities were added to SystemC in 2021, solving the long-standing pain point of requiring all ports to be bound.

## Core Concept / Everyday Analogy

### Empty Pins on a Circuit Board

Imagine a chip on a motherboard:

- Some pins are **unused** but can't be left floating, as that might cause undefined behavior
- **sc_unbound** = Connect the pin to an empty terminal (not connected to anything, but explicitly marked "intentionally unused")
- **sc_tie::value(0)** = Solder the pin directly to ground (fixed at 0)
- **sc_tie::value(1)** = Solder the pin directly to power (fixed at 1)

### Why is this needed?

In large SoC designs, an IP module might have 100 ports, but only 30 are used in a specific integration. Previously you had to manually create an `sc_signal` for each unused port and connect it, which was very tedious. Now you just need:

```cpp
module.unused_port(sc_unbound);           // Intentionally not connected
module.config_port(sc_tie::value(0x42));  // Bind to fixed value
```

## Detailed Class Descriptions

### `sc_stub<T>` - Stub Channel

```cpp
template <typename T>
class sc_stub : public sc_signal_inout_if<T>, public sc_prim_channel
```

A "fake" channel that implements the full signal interface but does nothing:

| Operation | Behavior |
|-----------|----------|
| `read()` | Returns initial value `m_init_val` |
| `write(val)` | **Ignored**, does nothing |
| `default_event()` | Returns an event that never fires |
| `value_changed_event()` | Same as above |
| `event()` | Always returns `false` |
| `posedge()` / `negedge()` | Always returns `false` |

### `sc_stub_registry` - Stub Management Center

```cpp
class sc_stub_registry
{
    void insert(sc_prim_channel* stub_);
    void clear();
};
```

All dynamically created `sc_stub` instances are registered here, with lifecycle managed by `sc_simcontext`. This prevents memory leaks -- since `sc_unbound` and `sc_tie::value()` use `new` to create stubs, someone needs to be responsible for deletion.

Safety checks:
- Cannot create stubs **during simulation execution**
- Cannot create stubs **after elaboration is complete**

### `sc_unbound` - Unbound Marker

```cpp
static sc_unbound_impl const sc_unbound = {};
```

A global static object. Its magic lies in the type conversion operator:

```cpp
template <typename T>
operator sc_signal_inout_if<T>&() const
{
    sc_stub<T>* stub = new sc_stub<T>(sc_gen_unique_name("sc_unbound"));
    sc_get_curr_simcontext()->get_stub_registry()->insert(...);
    return *stub;
}
```

When you write `port(sc_unbound)`:
1. The compiler automatically deduces the type `T`
2. Creates a new `sc_stub<T>`
3. Registers it with the stub registry
4. Returns an interface reference, completing the binding

**Limitation**: Can only be used with `sc_inout` or `sc_out` type ports (output/bidirectional), not with pure input ports `sc_in`, because the read value would be undefined.

### `sc_tie::value()` - Fixed Value Binding

```cpp
namespace sc_tie {
    template <typename T>
    sc_signal_in_if<T>& value(const T& val);
}
```

```cpp
sc_stub<T>* stub = new sc_stub<T>(sc_gen_unique_name("sc_tie::value"), val);
```

Similar to `sc_unbound`, but:
- A fixed value can be specified
- Returns `sc_signal_in_if<T>&`, so it **can be used with `sc_in` ports**
- Creates a new stub for each call

```mermaid
graph TD
    subgraph "sc_unbound Usage Flow"
        UB["port(sc_unbound)"] --> TC["Type conversion<br/>operator sc_signal_inout_if<T>&()"]
        TC --> NEW1["new sc_stub<T>(\"sc_unbound_0\")"]
        NEW1 --> REG1["Register with stub_registry"]
        REG1 --> BIND1["Binding complete"]
    end
    subgraph "sc_tie::value Usage Flow"
        TV["port(sc_tie::value(42))"] --> FN["sc_tie::value(42)"]
        FN --> NEW2["new sc_stub<T>(\"sc_tie::value_0\", 42)"]
        NEW2 --> REG2["Register with stub_registry"]
        REG2 --> BIND2["Binding complete<br/>read() always returns 42"]
    end
```

## Member Variables

### `sc_stub<T>`

| Variable | Type | Description |
|----------|------|-------------|
| `m_init_val` | `T` | Initial/fixed value, returned by `read()` |
| `ev` | `sc_event` | An event that never fires |

### `sc_stub_registry`

| Variable | Type | Description |
|----------|------|-------------|
| `m_stub_vec` | `std::vector<sc_prim_channel*>` | All registered stubs |
| `m_simc` | `sc_simcontext*` | Owning simulation context |

## Design Rationale / RTL Background

### Tie-Off in Hardware

In ASIC design, "tie-off" is a common practice:

- **Tie-high**: Connect unused input to VDD (power, logic 1)
- **Tie-low**: Connect unused input to GND (ground, logic 0)
- **NC** (No Connect): Not connected (but explicitly marked)

`sc_tie::value()` and `sc_unbound` simulate exactly these hardware design practices.

### Naming Convention

- Stubs created by `sc_unbound` are named `sc_unbound_0`, `sc_unbound_1`, ...
- Stubs created by `sc_tie::value()` are named `sc_tie::value_0`, `sc_tie::value_1`, ...

`sc_gen_unique_name` is used to ensure each stub has a unique name.

### Memory Management

All stubs are owned by `sc_stub_registry` and deleted in bulk via `clear()` when `sc_simcontext` is destroyed. This is a simple "arena allocation" pattern.

## Related Files

- `sc_signal_ifs.h` - Signal interface definitions (`sc_signal_in_if`, `sc_signal_inout_if`)
- `sc_prim_channel.h` - Primitive channel base class
- `sc_simcontext.h` - Simulation context (manages stub registry)
- `sc_communication_ids.h` - Error message IDs
