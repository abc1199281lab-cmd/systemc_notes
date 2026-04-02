# sc_clock_ports -- Type Aliases for Clock-Specific Ports

## Overview

`sc_clock_ports.h` defines three type aliases that provide semantically clearer port names for clock signals. These aliases exist purely for backward compatibility and code readability; under the hood, they are simply `bool`-typed signal ports.

**Source file:** `sc_clock_ports.h` (header-only)

## Everyday Analogy

Just like in everyday life, "alarm clock" and "wall clock" are both "timekeeping devices" but have different names for different purposes. `sc_in_clk` and `sc_in<bool>` are exactly the same thing, except the name tells the reader "this is connected to a clock signal".

## Definition

```cpp
typedef sc_in<bool>    sc_in_clk;
typedef sc_inout<bool> sc_inout_clk;
typedef sc_out<bool>   sc_out_clk;
```

It's that simple -- three `typedef` lines.

## Correspondence

| Clock Alias | Equivalent To | Purpose |
|-------------|---------------|---------|
| `sc_in_clk` | `sc_in<bool>` | Receive clock input (most common) |
| `sc_inout_clk` | `sc_inout<bool>` | Bidirectional clock (extremely rare) |
| `sc_out_clk` | `sc_out<bool>` | Clock output (extremely rare) |

## Conceptual Usage Example

```cpp
SC_MODULE(FlipFlop) {
    sc_in_clk clk;          // same as sc_in<bool> clk;
    sc_in<bool> d;
    sc_out<bool> q;

    void process() {
        q.write(d.read());
    }

    SC_CTOR(FlipFlop) {
        SC_METHOD(process);
        sensitive << clk.pos();  // trigger on clock positive edge
    }
};
```

## Design Notes

### Why not use template specialization?

These type aliases were introduced in the SystemC 2.0 era, when coding conventions favored `typedef`. They have been retained as a backward-compatibility mechanism so that legacy code does not need modification.

### When to use `sc_in_clk` vs `sc_in<bool>`?

Both are functionally identical. Recommendations:
- If the port is truly intended to receive a clock signal, use `sc_in_clk` for better readability
- If it's a general boolean signal (like reset, enable), use `sc_in<bool>`

## Related Files

- `sc_signal_ports.h` - Definitions of `sc_in<bool>` and related classes
- `sc_clock.h` - Clock channel, typically connected to `sc_in_clk`
