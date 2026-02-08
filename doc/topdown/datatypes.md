# Top-Down Analysis: Hardware Data Types

SystemC provides a rich set of data types to model hardware behavior, ranging from efficient native types to complex arbitrary-precision logic vectors.

## 1. The 4-Valued Logic System
Hardware modeling often requires representing states beyond just 0 and 1. SystemC typically uses a 4-valued logic system:
-   **'0' (Logic 0)**: Generating low voltage.
-   **'1' (Logic 1)**: Generating high voltage.
-   **'Z' (High Impedance)**: Not driving anything (floating). Crucial for bidirectional buses.
-   **'X' (Unknown)**: Conflict (multiple drivers driving different values) or uninitialized state.

### 1.1 `bool` vs `sc_logic`
-   **`bool`**: Standard C++ type (2-valued). Fast. Use for control logic or internal state where Z/X are impossible.
-   **`sc_logic`**: SystemC type (4-valued). Use for external ports (`sc_in<sc_logic>`) or signals where reset state or contention must be modeled.
-   **Note**: `sc_bit` is deprecated. Use `bool` or `sc_logic`.

## 2. Integer Types: Precision vs Speed
SystemC splits integer types based on the width requirement.

### 2.1 Limited Precision (`sc_int`, `sc_uint`)
-   **Width**: 1 to 64 bits.
-   **Speed**: **Fastest**. Maps directly to C++ `int64` / `uint64` with mask operations.
-   **Usage**: Control registers, counters, addresses within 64-bit space.
-   **Hardware Mapping**: Maps to `reg [W-1:0]` or `int` in Verilog.

### 2.2 Arbitrary Precision (`sc_bigint`, `sc_biguint`)
-   **Width**: Arbitrary (e.g., 1024 bits).
-   **Speed**: Slower. Requires dynamic memory allocation and software implementation of arithmetic (add-with-carry loops).
-   **Usage**: Wide data buses, crypto algorithms, huge accumulators.
-   **Hardware Mapping**: Maps to `reg [W-1:0]` in Verilog.

## 3. Vector Types: Bit vs Logic
Vectors allow manipulating bundles of bits.

### 3.1 Bit Vectors (`sc_bv`)
-   **Base**: `sc_bv_base`.
-   **Semantics**: Vector of **2-valued** bits (`0`, `1`).
-   **Usage**: Internal payloads, masks where Z/X logic is not needed. efficient than `sc_lv`.

### 3.2 Logic Vectors (`sc_lv`)
-   **Base**: `sc_lv_base`.
-   **Semantics**: Vector of **4-valued** logic (`0`, `1`, `X`, `Z`).
-   **Implementation**: Stores two bits for every logic value: one for data, one for control (to encode X/Z).
-   **Usage**: Bus signals, main data paths in RTL models.

## 4. Fixed Point (`sc_fixed`, `sc_ufixed`)
SystemC also supports fixed-point arithmetic (not detailed in this kernel analysis but present in `sysc/datatypes/fx`).
-   Allows bit-accurate modeling of DSP algorithms (integer bits + fractional bits).
-   Supports various quantization (rounding) and overflow (saturation) modes.

## 5. Summary Table

| Type | Values | Max Width | Speed | Best For |
| :--- | :--- | :--- | :--- | :--- |
| `bool` | 2 | 1 | Fast | Control Flags |
| `sc_logic` | 4 | 1 | Med | Ports, Tristate |
| `sc_int` / `sc_uint` | Integer | 64 | Fast | Arithmetic <= 64b |
| `sc_bigint` | Integer | Unlimited | Slow | Wide Arithmetic |
| `sc_bv` | 2 | Unlimited | Med | fast bit manipulation |
| `sc_lv` | 4 | Unlimited | Slow | RTL Buses |
