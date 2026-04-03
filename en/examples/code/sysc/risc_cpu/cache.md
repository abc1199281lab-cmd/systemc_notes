# Cache -- Instruction Cache and Data Cache

## Software Analogy

The role of cache in hardware is the same as a cache layer in software architecture (Redis, Memcached). The core concept is identical:

```
CPU <-> L1 Cache <-> Main Memory
App <-> Redis    <-> PostgreSQL
```

Accessing cache is much faster than accessing main memory. In this example, ICache (instruction cache) and DCache (data cache) are separate, an architecture known as **Harvard Architecture** -- like your application using different caches for code and data.

## Source Files

- `icache.h` / `icache.cpp` -- Instruction cache
- `dcache.h` / `dcache.cpp` -- Data cache

---

## ICache -- Instruction Cache

### Module Interface

| Direction | Signal Name | Type | Description |
|-----------|-------------|------|-------------|
| Input | `datain` | `sc_in<unsigned>` | Write data (for self-modifying code) |
| Input | `cs` | `sc_in<bool>` | Chip Select (enable) |
| Input | `we` | `sc_in<bool>` | Write Enable |
| Input | `addr` | `sc_in<unsigned>` | Access address |
| Input | `ld_valid` | `sc_in<bool>` | Process ID load valid |
| Input | `ld_data` | `sc_in<signed>` | Process ID value |
| Output | `dataout` | `sc_out<unsigned>` | Read instruction data |
| Output | `icache_valid` | `sc_out<bool>` | Output valid |
| Output | `stall_fetch` | `sc_out<bool>` | Stall Fetch |

### Internal Structure

```cpp
unsigned *icmemory;       // Instruction data array (up to 500 entries)
unsigned *ictagmemory;    // Tag memory (records which addresses are in cache)
signed int pid;           // Current Process ID
```

- Initialized by reading instruction contents from `icache.img`
- Uninitialized locations are filled with `0xeeeeeeee` (easy to identify during debugging)

### Behavioral Logic

```
while true:
    wait for cs == true
    if address >= BOOT_LENGTH (5):   # Only active after boot
        if ld_valid:
            update Process ID
        if we == true:
            write to icmemory[address]    # Self-Modifying Code (SMC)
        else:
            read icmemory[address]
            set icache_valid = true
    # Requests with address < 5 are handled by BIOS
```

### Important Design Decisions

- **BOOT_LENGTH = 5**: The first 5 instructions are provided by BIOS; ICache only handles requests with addresses >= 5.
- **Bounds checking**: When the address exceeds `MAX_CODE_LENGTH` (500), `0xFFFFFFFF` is output and a warning is printed.
- **Process ID support**: PID is loaded via `ld_valid` and `ld_data`, supporting cache switching in a multi-process environment.

---

## DCache -- Data Cache

### Module Interface

| Direction | Signal Name | Type | Description |
|-----------|-------------|------|-------------|
| Input | `datain` | `sc_in<signed>` | Write data |
| Input | `statein` | `sc_in<unsigned>` | MESI state bits |
| Input | `cs` | `sc_in<bool>` | Chip Select |
| Input | `we` | `sc_in<bool>` | Write Enable |
| Input | `addr` | `sc_in<unsigned>` | Access address |
| Input | `dest` | `sc_in<unsigned>` | Destination register |
| Output | `dataout` | `sc_out<signed>` | Read data |
| Output | `destout` | `sc_out<unsigned>` | Destination register (pass-through) |
| Output | `out_valid` | `sc_out<bool>` | Output valid |
| Output | `stateout` | `sc_out<unsigned>` | MESI state output |

### Internal Structure

```cpp
unsigned *dmemory;       // Data array (4000 entries)
unsigned *dsmemory;      // MESI state array
unsigned *dtagmemory;    // Tag memory
```

- Initialized by reading data from `dcache.img`
- Uninitialized locations are filled with `0xdeadbeef` (a classic debug magic number)

### MESI Protocol

DCache supports MESI cache coherence protocol state tagging:

| State Value | Code | Meaning | Software Analogy |
|-------------|------|---------|------------------|
| 3 | M (Modified) | Modified, inconsistent with main memory | Local unpushed changes |
| 2 | E (Exclusive) | Exclusive, only this cache has a copy | Local branch not forked |
| 1 | S (Shared) | Shared, multiple caches have copies | Multiple readers of the same data |
| 0 | I (Invalid) | Invalid | Cache miss / stale data |

In multi-core systems, MESI ensures all CPUs see a consistent memory state, analogous to cache invalidation strategies in distributed systems.

### Behavioral Logic

- **Write**: Data, state, and tag are written to the corresponding address simultaneously
- **Read**: Output data and MESI state, set `out_valid=true`, then clear after one cycle

## Comparison

| Feature | ICache | DCache |
|---------|--------|--------|
| Initialization source | `icache.img` | `dcache.img` |
| Capacity | 500 entries | 4000 entries |
| Default fill value | `0xeeeeeeee` | `0xdeadbeef` |
| Write support | Yes (SMC) | Yes |
| MESI support | No | Yes |
| Process ID | Yes | No |

## SystemC Key Points

- Both use `SC_CTHREAD`, driven on the rising clock edge.
- `do { wait(); } while (!(cs == true))` is the standard "wait to be activated" pattern.
- Cache memory is implemented with C++ dynamic arrays (`new unsigned[N]`) rather than `sc_signal` -- since this is the module's internal state and does not need cross-module communication.
