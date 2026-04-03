# Floating Point Unit -- FPU

## Software Analogy

The FPU is a dedicated coprocessor for floating point operations, equivalent to importing a specialized math library in software. In the era before CPUs integrated FPUs, floating point operations required a separate hardware chip -- like your web app delegating intensive computation to a Worker thread or microservice:

```python
# Software analogy
class FloatCoprocessor:
    def compute(self, op, a, b):
        # Unpack IEEE 754 format
        sign_a, exp_a, mant_a = self.unpack(a)
        sign_b, exp_b, mant_b = self.unpack(b)
        # Align exponents
        mant_a, mant_b, exp = self.align(exp_a, mant_a, exp_b, mant_b)
        # Perform operation
        result_mant = self.operate(op, mant_a, mant_b)
        # Repack
        return self.pack(sign_a, exp, result_mant)
```

## Source Files

- `floating.h` -- Module declaration
- `floating.cpp` -- Behavioral implementation

## Module Interface

| Direction | Signal Name | Type | Description |
|-----------|-------------|------|-------------|
| Input | `in_valid` | `sc_in<bool>` | FPU enable |
| Input | `opcode` | `sc_in<int>` | Operation code |
| Input | `floata` | `sc_in<signed int>` | Operand A (IEEE 754 format) |
| Input | `floatb` | `sc_in<signed int>` | Operand B (IEEE 754 format) |
| Input | `dest` | `sc_in<unsigned>` | Destination register number |
| Output | `fdout` | `sc_out<signed int>` | Computation result |
| Output | `fout_valid` | `sc_out<bool>` | Output valid |
| Output | `fdestout` | `sc_out<unsigned>` | Destination register number |

## IEEE 754 Floating Point Format

The FPU manually unpacks and assembles IEEE 754 single-precision format:

```
[31]     sign        (1 bit)   -- Sign bit (0=positive, 1=negative)
[30:23]  exponent    (8 bits)  -- Exponent
[22:0]   significand (23 bits) -- Significand (mantissa)
```

In software, you typically use `float` or `double` types directly, and the IEEE 754 details are hidden by the language. But in hardware, the FPU must implement every step itself.

## Operation Flow

1. **Unpack**: Extract sign, exponent, and significand from the 32-bit integer
2. **Exponent Alignment**: Right-shift the significand of the smaller exponent so both operands share the same exponent
3. **Compute**: Perform add/subtract/multiply/divide on the aligned significands
4. **Normalize**: Handle overflow and repack into IEEE 754 format

### Supported Operations

| Op Code | Operation | Description |
|---------|-----------|-------------|
| 0 | Stall | Hold previous value |
| 3 | FADD | Floating point addition |
| 4 | FSUB | Floating point subtraction |
| 5 | FMUL | Floating point multiplication (exponent doubling) |
| 6 | FDIV | Floating point division |

## Limitations

The source code comments explicitly state the following limitations:
- Correct results only when both operands have the same sign bit
- Overflow handling is simplified (overflow is ignored)

These limitations are acceptable in a teaching example -- a complete IEEE 754 implementation would be much more complex.

## Timing Characteristics

- Initialization waits 3 clock cycles
- Waits for the `in_valid` signal
- Exponent alignment takes 1 clock cycle
- Actual computation takes 1 clock cycle
- After outputting the result, waits 1 clock cycle before clearing the valid signal

A single floating point operation takes approximately 4-5 clock cycles overall, reflecting the reality that floating point operations are slower than integer operations.

## SystemC Key Points

- Uses the `do { wait(); } while (!(in_valid == true))` pattern to wait for input, a common SystemC synchronization pattern.
- The FPU and ALU are parallel execution units; Decode selects which one to activate via different valid signals.
- The output signals `fdout`, `fout_valid`, `fdestout` are shared with the MMX unit (`SC_MANY_WRITERS` policy), since both never operate simultaneously.
