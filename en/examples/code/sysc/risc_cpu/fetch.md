# Fetch -- Instruction Fetch Unit

## Software Analogy

The Fetch unit's job is like the "read the next line of code" function in a script interpreter. Imagine you are implementing a simple bytecode VM:

```python
# Software analogy: a simple instruction fetch loop
pc = 0
while True:
    instruction = memory[pc]    # <-- This is what Fetch does
    pc += 1
    if branch_taken:
        pc = branch_target
    if interrupt_pending:
        pc = interrupt_vector
    yield instruction
```

The difference is that the hardware version must handle memory latency, cache hit/miss, and timing synchronization with other modules.

## Source Files

- `fetch.h` -- Module declaration (interface definition)
- `fetch.cpp` -- Behavioral implementation

## Module Interface

### Input Signals

| Signal Name | Type | Description |
|-------------|------|-------------|
| `ramdata` | `sc_in<unsigned>` | Instruction data read from memory |
| `branch_address` | `sc_in<unsigned>` | Branch jump target address |
| `next_pc` | `sc_in<bool>` | Whether to increment PC |
| `branch_valid` | `sc_in<bool>` | Whether the branch is valid |
| `stall_fetch` | `sc_in<bool>` | Stall instruction fetch (pipeline stall) |
| `interrupt` | `sc_in<bool>` | Interrupt request |
| `int_vectno` | `sc_in<unsigned>` | Interrupt vector number |
| `bios_valid` | `sc_in<bool>` | BIOS data ready |
| `icache_valid` | `sc_in<bool>` | ICache data ready |
| `pred_fetch` | `sc_in<bool>` | Branch prediction triggered |
| `pred_branch_address` | `sc_in<unsigned>` | Predicted branch target address |
| `pred_branch_valid` | `sc_in<bool>` | Whether branch prediction is valid |

### Output Signals

| Signal Name | Type | Description |
|-------------|------|-------------|
| `ram_cs` | `sc_out<bool>` | Memory chip select (enable memory) |
| `ram_we` | `sc_out<bool>` | Memory write enable |
| `address` | `sc_out<unsigned>` | Address sent to memory |
| `instruction` | `sc_out<unsigned>` | Instruction sent to Decode |
| `instruction_valid` | `sc_out<bool>` | Instruction valid |
| `program_counter` | `sc_out<unsigned>` | Current PC value |
| `interrupt_ack` | `sc_out<bool>` | Interrupt acknowledgment |
| `branch_clear` | `sc_out<bool>` | Clear pending branch |
| `reset` | `sc_out<bool>` | Reset signal |

## Behavioral Logic

Fetch uses `SC_CTHREAD` (clocked thread), triggered on each rising clock edge.

### Boot Sequence

1. Assert `reset=true`, enable memory read (`ram_cs=true, ram_we=false`)
2. Wait `memory_latency` cycles for data to appear
3. Wait for `bios_valid` or `icache_valid` to become true
4. Read the first instruction and send it to Decode
5. When PC reaches 5, deassert reset (BIOS phase ends, switch to ICache)

### Main Loop

```
while (true):
    if interrupt occurred:
        PC = interrupt vector address
        read first instruction of interrupt handler
        send interrupt acknowledgment
    else if branch valid:
        PC = branch target address
        read instruction from new address
        set branch_clear
    else:
        read instruction from current PC
        PC++
```

### Key Observations

- **Stall mechanism**: When `stall_fetch` is true, the read data is discarded (set to 0), similar to a backpressure mechanism in software that pauses the producer.
- **Memory latency simulation**: `wait(memory_latency)` simulates real memory access latency, equivalent to `sleep()` or `await` in software simulation.
- **Interrupts take priority over branches**: The interrupt check comes before the branch check, ensuring hardware interrupts are handled promptly.

## SystemC Key Points

- Uses `SC_CTHREAD` with `CLK.pos()`, meaning this behavior is driven on each rising clock edge.
- `wait()` suspends execution until the next clock edge, similar to `await nextTick()`.
- `do { wait(); } while (condition)` is a common "wait for condition" pattern, similar to busy-wait but consuming no resources beyond simulation time.
