# sc_fifo_ports.h - FIFO-Specific Port Classes

## Overview

This file defines two port classes, `sc_fifo_in<T>` and `sc_fifo_out<T>`, for connecting to the read end and write end of a FIFO channel respectively. They wrap `sc_port` and provide more convenient access methods, allowing modules to call `read()` / `write()` directly without using the `->` operator.

## Core Concept / Everyday Analogy

### Pipe Fittings

Imagine two pipes connecting to a water tower (FIFO):

- **Inlet fitting** (`sc_fifo_in`): Can only draw water from the tower, connects to the tower's outlet
- **Outlet fitting** (`sc_fifo_out`): Can only pour water into the tower, connects to the tower's inlet
- The fitting specification (interface) ensures it can only connect to a FIFO-type tower, not to a faucet (regular signal)

```mermaid
graph LR
    Producer["Producer Module"] -->|sc_fifo_out| FIFO["sc_fifo Channel"]
    FIFO -->|sc_fifo_in| Consumer["Consumer Module"]
```

## Detailed Class Descriptions

### `sc_fifo_in<T>` - FIFO Input Port

```cpp
template <class T>
class sc_fifo_in
: public sc_port<sc_fifo_in_if<T>, 0, SC_ONE_OR_MORE_BOUND>
```

#### Type Definitions

| Type Alias | Actual Type | Description |
|------------|------------|-------------|
| `data_type` | `T` | Data type |
| `if_type` | `sc_fifo_in_if<T>` | Bound interface type |
| `base_type` | `sc_port<...>` | Base port type |

#### Shortcut Methods

These methods forward directly to the underlying interface, eliminating the `(*this)->` syntax:

| Method | Description |
|--------|-------------|
| `read(data_type&)` | Blocking read |
| `read()` | Blocking read (return value) |
| `nb_read(data_type&)` | Non-blocking read |
| `num_available()` | Number of readable samples |
| `data_written_event()` | Data written event |

#### Event Finder

```cpp
sc_event_finder& data_written() const;
```

Used for **static sensitivity** setup. For example:

```cpp
SC_METHOD(process);
sensitive << fifo_in.data_written();
```

When the FIFO has new data written, `process` will be triggered. Internally uses `sc_event_finder::cached_create` to cache-create the event finder.

### `sc_fifo_out<T>` - FIFO Output Port

```cpp
template <class T>
class sc_fifo_out
: public sc_port<sc_fifo_out_if<T>, 0, SC_ONE_OR_MORE_BOUND>
```

Symmetric structure to `sc_fifo_in`:

| Method | Description |
|--------|-------------|
| `write(const data_type&)` | Blocking write |
| `nb_write(const data_type&)` | Non-blocking write |
| `num_free()` | Number of writable free slots |
| `data_read_event()` | Data read event |
| `data_read()` | Event finder for static sensitivity |

## Constructor Overview

Both classes provide multiple constructors supporting different creation methods:

| Creation Method | Description |
|-----------------|-------------|
| Default construction | Auto-named |
| `const char* name_` | Named |
| `in_if_type& interface_` | Direct interface binding |
| `in_port_type& parent_` | Bind to parent port |
| `this_type& parent_` | Bind to same-type parent port |
| Name + interface/parent combinations | Named versions of the above |

## Design Rationale

### Why not just use `sc_port`?

1. **Convenience**: `fifo_in.read()` is more intuitive than `fifo_in->read()`
2. **Static sensitivity support**: `data_written()` / `data_read()` event finders are FIFO-specific
3. **Type safety**: Ensures connection only to FIFO interfaces

### `SC_ONE_OR_MORE_BOUND` Binding Policy

This means the port must bind to at least one channel. Although the second template parameter of `sc_port` is set to 0 (meaning no upper limit), combined with `SC_ONE_OR_MORE_BOUND` it ensures no unbound ports.

### Event Finder Caching

`m_written_finder_p` / `m_read_finder_p` use the `mutable` qualifier because event finder creation is lazy and the pointer needs to be modified even in `const` methods. The destructor is responsible for deletion.

## Related Files

- `sc_fifo.h` - FIFO channel implementation
- `sc_fifo_ifs.h` - FIFO interface definitions
- `sc_port.h` - Generic port base class
- `sc_signal_ports.h` - Signal ports (similar but for signal channels)
