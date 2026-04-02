# sc_trace.h / sc_trace.cpp - Signal Tracing Public API

> Defines the abstract interface `sc_trace_file` for trace files, and the global `sc_trace()` function family, allowing users to record signals of various data types into trace files.

## Everyday Analogy

Imagine you have a "universal camera" that can record anything you give it (numbers, booleans, logic signals). `sc_trace()` is this camera's "capture button" -- you just tell it "which variable to capture" and "what name to give it", and it will record the entire history of that variable's changes.

`sc_trace_file` is the camera's "interface specification" -- it only defines what functions the camera must have (recording bool, int, float...), regardless of what film you use internally (VCD or WIF format).

## Overview

This file is the **public entry point** of the tracing subsystem. It does two things:

1. **Defines the abstract base class `sc_trace_file`**: declares all pure virtual functions that every trace file must implement
2. **Provides global `sc_trace()` function overloads**: offers a convenient calling interface for each data type

## sc_trace_file Abstract Class

### Class Definition

```cpp
class sc_trace_file
{
    friend class sc_simcontext;
public:
    sc_trace_file();

    // Pure virtual trace methods for each data type
    virtual void trace(const bool& object, const std::string& name) = 0;
    virtual void trace(const int& object, const std::string& name, int width) = 0;
    // ... (many more overloads)

    virtual void trace(const unsigned int& object,
                       const std::string& name,
                       const char** enum_literals) = 0;

    virtual void write_comment(const std::string& comment) = 0;
    virtual void space(int n);
    virtual void delta_cycles(bool flag);
    virtual void set_time_unit(double v, sc_time_unit tu) = 0;

protected:
    virtual void cycle(bool delta_cycle) = 0;
    const sc_dt::uint64& event_trigger_stamp(const sc_event& event) const;
    virtual ~sc_trace_file() {}
};
```

### Trace Method Categories

Trace methods are expanded through macros and fall into two categories:

| Macro | Parameter Form | Applicable Types |
|-------|---------------|-----------------|
| `DECL_TRACE_METHOD_A(tp)` | `(const tp& object, const string& name)` | `bool`, `float`, `double`, `sc_logic`, `sc_signed`, etc. |
| `DECL_TRACE_METHOD_B(tp)` | `(const tp& object, const string& name, int width)` | `char`, `short`, `int`, `long`, `int64`, `uint64` and other integer types |

**Why two categories?** Type A covers types with "fixed width or self-contained width information" (e.g., `bool` is 1 bit, `sc_signed` knows its own width). Type B covers "C++ native integers", whose width may vary across platforms, so an additional `width` parameter is needed.

### Key Method Descriptions

| Method | Description |
|--------|-------------|
| `trace(...)` | Register an object for tracing; each type has a corresponding pure virtual version |
| `write_comment(comment)` | Write a comment into the trace file |
| `space(n)` | Set field spacing (not used by most formats, default empty implementation) |
| `delta_cycles(flag)` | Whether to trace changes between delta cycles |
| `set_time_unit(v, tu)` | Set the time unit of the trace file |
| `cycle(delta_cycle)` | [protected] Called each time the simulation advances, writes trace data for that moment |
| `event_trigger_stamp(event)` | [protected] Get the trigger timestamp of an event, used by subclasses to trace events |

## Global sc_trace() Functions

### Function Overload Strategy

Global `sc_trace()` functions are also expanded through macros, providing two versions for each type:

```cpp
// Reference version
void sc_trace(sc_trace_file* tf, const bool& object, const std::string& name);

// Pointer version
void sc_trace(sc_trace_file* tf, const bool* object, const std::string& name);
```

The pointer version is simply a wrapper around the reference version that automatically dereferences.

### Implementation Logic

All global `sc_trace()` function implementations are very simple -- first check if `tf` is null, then forward to `tf->trace()`:

```cpp
void sc_trace(sc_trace_file* tf, const bool& object, const std::string& name)
{
    if (tf) {
        tf->trace(object, name);
    }
}
```

### Signal Interface Template Specialization

For `sc_signal_in_if<T>`, a template version is provided that directly reads the signal value for tracing:

```cpp
template <class T>
void sc_trace(sc_trace_file* tf,
              const sc_signal_in_if<T>& object,
              const std::string& name)
{
    sc_trace(tf, object.read(), name);
}
```

Specialized versions requiring a `width` parameter are also provided for signal interfaces of `char`, `short`, `int`, and `long`.

### Helper Functions

| Function | Description |
|----------|-------------|
| `sc_trace_delta_cycles(tf, on)` | Toggle delta cycle tracing, inline wrapper for `tf->delta_cycles(on)` |
| `sc_write_comment(tf, comment)` | Write a comment, inline wrapper for `tf->write_comment(comment)` |
| `tprintf(tf, format, ...)` | `printf`-like formatted write to trace comment |
| `sc_create_vcd_trace_file(name)` | Create a VCD format trace file |
| `sc_close_vcd_trace_file(tf)` | Close a VCD trace file |
| `sc_create_wif_trace_file(name)` | Create a WIF format trace file |
| `sc_close_wif_trace_file(tf)` | Close a WIF trace file |

### void* Dummy Version

```cpp
void sc_trace(sc_trace_file* tf, const void* object, const std::string& name);
```

This version is a "catch-all" that gets selected when the C++ compiler cannot find a suitable overload. It **does nothing**, only emits a `SC_ID_TRACING_OBJECT_IGNORED_` warning telling the user "this type cannot be traced".

### Enum Tracing (Deprecated)

```cpp
void sc_trace(sc_trace_file* tf, const unsigned int& object,
              const std::string& name, const char** enum_literals);
```

This is a deprecated feature from the IEEE 1666 standard, used to trace enum values and write string literals to the trace file. A deprecation warning is issued on the first call.

## Design Decisions

### Why Use Macro Expansion?

The number of types that need tracing support is enormous (`bool`, `char`, `short`, `int`, `long`, `int64`, `uint64`, `float`, `double`, `sc_bit`, `sc_logic`, `sc_signed`, `sc_unsigned`, etc.), and the trace function signatures for each type are almost identical, differing only in the type name. Macro expansion avoids massive code duplication while ensuring every type has complete declarations and definitions.

### Why Is sc_trace_file a Friend of sc_simcontext?

Because the `cycle()` method is `protected` -- only the simulation kernel should decide "when to record a data point"; users should not manually call `cycle()`.

## Usage Example

```cpp
// Create trace file
sc_trace_file* tf = sc_create_vcd_trace_file("waveform");

// Set time unit
tf->set_time_unit(1, SC_NS);

// Register signals to trace
sc_signal<bool> clk;
sc_signal<int> data;
sc_trace(tf, clk, "clk");
sc_trace(tf, data, "data");

// After simulation
sc_close_vcd_trace_file(tf);
```

## Related Files

- [sc_trace_file_base.md](sc_trace_file_base.md) -- Shared implementation base for `sc_trace_file`
- [sc_vcd_trace.md](sc_vcd_trace.md) -- Concrete VCD format implementation
- [sc_wif_trace.md](sc_wif_trace.md) -- Concrete WIF format implementation
- [sc_tracing_ids.md](sc_tracing_ids.md) -- Tracing-related error message IDs
- `sysc/communication/sc_signal_ifs.h` -- Signal interface `sc_signal_in_if<T>`
- `sysc/kernel/sc_event.h` -- Event class, provides `m_trigger_stamp`
