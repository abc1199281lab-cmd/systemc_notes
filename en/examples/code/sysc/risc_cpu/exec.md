# Execute -- Integer Execution Unit (ALU)

## Software Analogy

The Execute unit is the `eval()` function. It receives the already-parsed opcode and operands, then performs the corresponding operation:

```python
# Software analogy
def alu_execute(opcode, a, b):
    match opcode:
        case 3:  return a + b
        case 4:  return a - b
        case 5:  return a * b
        case 6:  return a // b
        case 7:  return ~(a & b)
        case 8:  return a & b
        case 9:  return a | b
        case 10: return a ^ b
        case 11: return ~a
        case 14: return a % b
```

It is that straightforward. The complexity is in Decode; Execute is responsible only for pure computation.

## Source Files

- `exec.h` -- Module declaration
- `exec.cpp` -- Behavioral implementation

## Module Interface

### Input Signals

| Signal Name | Type | Description |
|-------------|------|-------------|
| `in_valid` | `sc_in<bool>` | Decode output valid |
| `opcode` | `sc_in<int>` | ALU operation code |
| `dina` | `sc_in<signed int>` | Operand A |
| `dinb` | `sc_in<signed int>` | Operand B |
| `dest` | `sc_in<unsigned>` | Destination register number |
| `forward_A` / `forward_B` | `sc_in<bool>` | Data forwarding flags (unused) |

### Output Signals

| Signal Name | Type | Description |
|-------------|------|-------------|
| `dout` | `sc_out<signed int>` | Computation result |
| `out_valid` | `sc_out<bool>` | Output valid |
| `destout` | `sc_out<unsigned>` | Destination register number (passed to Decode for writeback) |
| `C` | `sc_out<bool>` | Carry flag |
| `V` | `sc_out<bool>` | Overflow flag |
| `Z` | `sc_out<bool>` | Zero flag |

## Supported Operations

| Op Code | Operation | Latency (clock cycles) | Description |
|---------|-----------|------------------------|-------------|
| 0 | Stall | 1 | Hold previous output value |
| 1 | Add + Carry | 1 | a + b + carry |
| 2 | Sub + Carry | 1 | a - b - carry |
| 3 | Add | 1 | a + b |
| 4 | Sub | 1 | a - b |
| 5 | Multiply | 2 | a * b (one extra cycle) |
| 6 | Divide | 2 | a / b (one extra cycle, with division-by-zero check) |
| 7 | NAND | 1 | ~(a & b) |
| 8 | AND | 1 | a & b |
| 9 | OR | 1 | a \| b |
| 10 | XOR | 1 | a ^ b |
| 11 | NOT | 1 | ~a |
| 12 | Left Shift | 1 | a << b |
| 13 | Right Shift | 1 | a >> b |
| 14 | Modulo | 1 | a % b |

## Condition Flags

After each operation, the ALU updates three condition flags:

- **Z (Zero)**: Set when the result is zero. Similar to a `if (result == 0)` boolean state.
- **C (Carry)**: Triggered when bit 32 of the result is set, indicating unsigned overflow.
- **V (Overflow)**: Triggered when the result exceeds the 32-bit range, indicating signed overflow.

In software, these flags are equivalent to global state variables that are automatically updated after each operation, used by subsequent branch instructions.

## Timing Characteristics

- Most operations take 1 clock cycle
- Multiplication and division take 2 clock cycles (simulating the fact that these operations are slower in real hardware)
- `wait(3)` before the loop begins means the Execute unit waits 3 cycles after initialization before starting, ensuring the earlier pipeline stages have prepared data

## SystemC Key Points

- Uses `sc_dt::int64` for intermediate results to detect 32-bit overflow. This is similar to using BigInt in JavaScript to detect Number overflow.
- The output is set to invalid (`out_valid=false`) after one `wait()`, ensuring Decode only sees a single cycle of valid signal -- this is a common pulse signal pattern.
