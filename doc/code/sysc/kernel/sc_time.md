# SystemC Time Analysis

> **File**: `ref/systemc/src/sysc/kernel/sc_time.cpp`

## 1. Overview
`sc_time` represents simulation time. In the kernel, time is not a floating-point number but a **64-bit unsigned integer** count of "time steps".

## 2. Core Concepts

### 2.1 Time Resolution (`sc_time_params`)
SystemC enforces a global time resolution (smallest representable time unit).
- **Default**: 1 ps (`1e-12` s).
- **Storage**: `m_value` in `sc_time` is the count of these resolution steps.
- **Example**: If resolution is 1ns, `sc_time(10, SC_NS)` stores `10`. If resolution is 1ps, it stores `10,000`.

### 2.2 Construction & Conversion
When you write `sc_time(10.5, SC_NS)`:
1.  It retrieves the global time resolution.
2.  It converts `10.5 ns` into the number of resolution steps.
3.  **Rounding**: It rounds to the nearest integer. `10.5 * scaling_factor + 0.5`.

### 2.3 The `time_params` Lifecycle
- `sc_time_params` is a singleton structure in `sc_simcontext`.
- **Locking**: The resolution is "frozen" (`time_resolution_fixed`) as soon as the *first* `sc_time` object is created.
    - *Implication*: You must call `sc_set_time_resolution()` *before* declaring any `sc_time` constants.

### 2.4 String Conversion
Logic exists to parse strings like `"10 ns"`, `"5.5 us"`. This is often used for reading configuration files or command-line arguments.

---

## 3. Hardware / RTL Mapping

| SystemC (`sc_time`) | Verilog / SystemVerilog |
| :--- | :--- |
| `sc_time` | `#` delay values |
| `sc_set_time_resolution()` | `timescale` (the precision part) |
| `sc_start(time)` | `initial #time $finish;` |

- **Precision**: Unlike Verilog `timescale` which can vary per module, SystemC has a **single global resolution**. This prevents the classic Verilog issue of precision loss when connecting modules with different timescales.

---

## 4. Key Takeaways
1.  **Integer Arithmetic**: Internally, time is `uint64`. This avoids floating-point drift over long simulations.
2.  **Global Resolution**: Set it once, early. If you need femtosecond precision, set it to `SC_FS` before creating any objects.
3.  **Performance**: Using `SC_ZERO_TIME` often optimizes to a specific check, but creating many `sc_time` objects (conversions) has a small cost.
