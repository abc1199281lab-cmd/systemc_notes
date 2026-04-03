# MMX Unit -- SIMD Execution Unit

## Software Analogy

The MMX unit implements SIMD (Single Instruction, Multiple Data) operations. If you have used numpy, you already understand the core concept of SIMD:

```python
# Software analogy: regular element-wise operations
for i in range(4):
    c[i] = a[i] + b[i]

# SIMD equivalent (numpy style)
c = a + b   # One operation, four elements computed simultaneously
```

In hardware, MMX splits a 32-bit register into four 8-bit bytes and independently performs the same operation on each byte. This is very useful in image processing and multimedia applications -- for example, adding four pixel values simultaneously.

## Source Files

- `mmxu.h` -- Module declaration
- `mmxu.cpp` -- Behavioral implementation

## Module Interface

| Direction | Signal Name | Type | Description |
|-----------|-------------|------|-------------|
| Input | `mmx_valid` | `sc_in<bool>` | MMX enable |
| Input | `opcode` | `sc_in<int>` | Operation code |
| Input | `mmxa` | `sc_in<signed int>` | Operand A (packed 4 bytes) |
| Input | `mmxb` | `sc_in<signed int>` | Operand B (packed 4 bytes) |
| Input | `dest` | `sc_in<unsigned>` | Destination register |
| Output | `mmxdout` | `sc_out<signed int>` | Computation result |
| Output | `mmxout_valid` | `sc_out<bool>` | Output valid |
| Output | `mmxdestout` | `sc_out<unsigned>` | Destination register number |

## Packed Byte Format

A 32-bit value is split into four 8-bit lanes:

```
[31:24] byte3 (a3)    [23:16] byte2 (a2)    [15:8] byte1 (a1)    [7:0] byte0 (a0)
```

This is equivalent to in software:

```python
a3 = (value >> 24) & 0xFF
a2 = (value >> 16) & 0xFF
a1 = (value >>  8) & 0xFF
a0 =  value        & 0xFF
```

## Supported Operations

| Op Code | Instruction | Description | Software Analogy |
|---------|-------------|-------------|------------------|
| 3 | PADD | Packed byte addition | `np.add(a, b)` (uint8) |
| 4 | PADDS | Saturating addition (capped at 255) | `np.clip(a + b, 0, 255)` |
| 5 | PSUB | Packed byte subtraction | `np.subtract(a, b)` |
| 6 | PSUBS | Saturating subtraction (floored at 0) | `np.clip(a - b, 0, 255)` |
| 7 | PMADD | Packed multiply-add | `a3*b3+a2*b2` and `a1*b1+a0*b0` |
| 8 | PACK | 16-bit to 8-bit packing | Data compression |
| 9 | MMXCK | Chroma Keying | Comparison mask (for green screen removal) |

### Saturation Arithmetic

Normal addition wraps around on overflow (e.g., 200 + 100 = 44, because 300 mod 256 = 44). Saturating addition clamps to the maximum value of 255 instead, which is important when processing pixel values to avoid color overflow causing visual artifacts.

### Chroma Keying

The `MMXCK` operation compares each byte channel of two pixels for equality: equal outputs 0xFF, not equal outputs 0x00. This is the fundamental operation for "green screen removal" in video compositing -- comparing whether each pixel matches a specific background color.

## Timing Characteristics

- All MMX operations take only 1 clock cycle
- The output signal `mmxdout` is shared with the FPU's `fdout` (via `SC_MANY_WRITERS`), since Decode guarantees that FPU and MMX are never activated simultaneously

## SystemC Key Points

- MMX and FPU write to the same output signals (`fdout`, `fout_valid`, `fdestout`), using the `SC_MANY_WRITERS` write policy to allow multiple writers.
- In `main.cpp`, the MMX outputs are connected directly to the same signal lines as the FPU; Decode receives results from both through the unified `fpu_valid` signal.
