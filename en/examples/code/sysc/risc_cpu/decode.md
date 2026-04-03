# Decode -- Instruction Decode Unit

## Software Analogy

The Decode unit is like a command parser. Imagine you are implementing a CLI tool or script interpreter:

```python
# Software analogy
def decode(raw_instruction):
    opcode = (raw_instruction >> 24) & 0xFF       # Extract operation code
    regC   = (raw_instruction >> 20) & 0xF        # Destination register
    regA   = (raw_instruction >> 16) & 0xF        # Source register A
    regB   = (raw_instruction >> 12) & 0xF        # Source register B
    imm    = raw_instruction & 0xFFFF             # Immediate value

    match opcode:
        case 0x01: return ("ADD", regC, regs[regA], regs[regB])
        case 0x04: return ("SUB", regC, regs[regA], regs[regB])
        case 0x10: return ("BEQ", regC, regA, imm)
        ...
```

Decode is the **most complex module** in this CPU because it simultaneously handles: instruction decoding, register reads, writeback data reception, and branch decisions.

## Source Files

- `decode.h` -- Module declaration
- `decode.cpp` -- Behavioral implementation

## Instruction Format

Each instruction is a 32-bit unsigned integer with the following format:

```
[31:24]  opcode       (8 bits)  -- Operation code
[23:20]  regC         (4 bits)  -- Destination register
[19:16]  regA         (4 bits)  -- Source register A
[15:12]  regB         (4 bits)  -- Source register B
[15:0]   immediate    (16 bits) -- Immediate value (overlaps with regB)
[11:0]   offset       (12 bits) -- Memory offset
[23:0]   longlabel    (24 bits) -- Long jump label
```

## Instruction Set Overview

### Integer Operations (sent to ALU)

| Opcode | Instruction | Description | ALU Op |
|--------|-------------|-------------|--------|
| 0x00 | HALT | Halt and dump registers | - |
| 0x01 | ADD Rc, Ra, Rb | Rc = Ra + Rb | 3 |
| 0x02 | ADDI Rc, Ra, #imm | Rc = Ra + imm | 3 |
| 0x03 | ADDC Rc, Ra, Rb | Rc = Ra + Rb + Carry | 1 |
| 0x04 | SUB Rc, Ra, Rb | Rc = Ra - Rb | 4 |
| 0x05 | SUBI Rc, Ra, #imm | Rc = Ra - imm | 4 |
| 0x06 | SUBC Rc, Ra, Rb | Rc = Ra - Rb - Carry | 2 |
| 0x07 | MUL Rc, Ra, Rb | Rc = Ra * Rb | 5 |
| 0x08 | DIV Rc, Ra, Rb | Rc = Ra / Rb | 6 |
| 0x09 | NAND Rc, Ra, Rb | Rc = ~(Ra & Rb) | 7 |
| 0x0A | AND Rc, Ra, Rb | Rc = Ra & Rb | 8 |
| 0x0B | OR Rc, Ra, Rb | Rc = Ra \| Rb | 9 |
| 0x0C | XOR Rc, Ra, Rb | Rc = Ra ^ Rb | 10 |
| 0x0D | NOT Rc, Ra | Rc = ~Ra | 11 |
| 0x0E | MOD Rc, Ra, Rb | Rc = Ra % Rb | 14 |
| 0x0F | MOV Rc, Ra | Rc = Ra | 3 |
| 0xF1 | MOVI Rc, #imm | Rc = imm | 3 |

### Branch Instructions

| Opcode | Instruction | Description |
|--------|-------------|-------------|
| 0x10 | BEQ Rc, Ra, label | if Rc == Ra then PC += label |
| 0x11 | BNE Rc, Ra, label | if Rc != Ra then PC += label |
| 0x12 | BGT Rc, Ra, label | if Rc > Ra then PC += label |
| 0x13 | BGE Rc, Ra, label | if Rc >= Ra then PC += label |
| 0x14 | BLT Rc, Ra, label | if Rc < Ra then PC += label |
| 0x15 | BLE Rc, Ra, label | if Rc <= Ra then PC += label |
| 0x16 | J longlabel | Unconditional jump |

### Memory Operations

| Opcode | Instruction | Description |
|--------|-------------|-------------|
| 0x4D | LW Rc, Ra, offset | Rc = mem[Ra + offset] |
| 0x4E | SW Rc, Ra, offset | mem[Ra + offset] = Rc |

### Floating Point Operations (sent to FPU)

| Opcode | Instruction | Description |
|--------|-------------|-------------|
| 0x29 | FADD | Floating point addition |
| 0x2A | FSUB | Floating point subtraction |
| 0x2B | FMUL | Floating point multiplication |
| 0x2C | FDIV | Floating point division |

### MMX/SIMD Operations (sent to MMX Unit)

| Opcode | Instruction | Description |
|--------|-------------|-------------|
| 0x31 | PADD | Packed byte addition |
| 0x32 | PADDS | Packed byte saturating addition |
| 0x33 | PSUB | Packed byte subtraction |
| 0x34 | PSUBS | Packed byte saturating subtraction |
| 0x35 | PMADD | Packed multiply-add |
| 0x36 | PACK | Data packing |
| 0x37 | MMXCK | Chroma keying |

### System Instructions

| Opcode | Instruction | Description |
|--------|-------------|-------------|
| 0xE0 | FLUSH | Clear all registers |
| 0xF0 | LDPID | Load Process ID |
| 0xFF | QUIT | End simulation |

## Internal State

```cpp
signed int cpu_reg[32];       // 32 general-purpose registers (like a HashMap<int, int>)
signed int vcpu_reg[32];      // Virtual registers (unused)
bool cpu_reg_lock[32];        // Register lock flags (for data hazard handling)
unsigned int pc_reg;          // Program Counter
unsigned int jalpc_reg;       // Return address register (for function calls)
```

The register file is loaded with initial values from `register.img` at construction time.

## Behavioral Logic

### Writeback Handling

At the beginning of each clock cycle, Decode first checks whether there is data to write back to registers:

1. **ALU result writeback**: If `destreg_write == true`, write the ALU output to `cpu_reg[destreg_write_src]`
2. **Memory load writeback**: If `dram_rd_valid == true`, write the DCache data to a register
3. **FPU/MMX result writeback**: If `fpu_valid == true`, write the floating point/MMX result to a register

This is like an event handler processing "replies" before "new requests" on each tick.

### Decoding and Dispatch

After reading an instruction, the fields are extracted via bit masking, then a large `switch` statement determines the action based on the opcode:

- **ALU instructions**: Set `src_A`, `src_B`, `alu_op`, `decode_valid=true`, send to Execute
- **FPU instructions**: Set the same operands but enable `float_valid=true`
- **MMX instructions**: Set the same operands but enable `mmx_valid=true`
- **Branch instructions**: Compare register values directly in the Decode stage; if the branch is taken, set `branch_valid=true` and `branch_target_address`
- **Memory instructions**: Set `mem_access=true` and `mem_address`

### Branch Handling

Branch decisions are made in the Decode stage (not Execute), which reduces branch latency. Backward branches are detected via the sign bit at bit 15, and the label is converted to a negative offset.

## SystemC Key Points

- The 32 registers are implemented as a C++ array and do not need `sc_signal`, since they are private state of the module.
- Multiple writeback sources (ALU, DCache, FPU) are handled sequentially within the same process, avoiding multiple-write conflicts.
- `sc_stop()` is called when the QUIT (0xFF) instruction is received, terminating the entire simulation.
