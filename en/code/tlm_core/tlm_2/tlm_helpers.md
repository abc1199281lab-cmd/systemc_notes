# tlm_helpers.h - Endianness Detection Helper Functions

## Overview

`tlm_helpers.h` provides a few simple helper functions for detecting the host machine's byte order (endianness) at runtime. These functions play a foundational role in endian conversion and cross-platform simulation.

## Everyday Analogy

Just like checking whether your destination uses 110V or 220V before traveling abroad -- these functions help you "detect" which byte ordering your computer uses, so you can decide whether "voltage conversion" (endian conversion) is needed.

## Enum

```cpp
enum tlm_endianness {
  TLM_UNKNOWN_ENDIAN,   // not yet determined
  TLM_LITTLE_ENDIAN,    // least significant byte first (x86, ARM default)
  TLM_BIG_ENDIAN        // most significant byte first (PowerPC, SPARC)
};
```

## Function Details

### `get_host_endianness()`

```cpp
inline tlm_endianness get_host_endianness(void);
```

Returns the host's endianness. Uses a static variable for caching -- detection only occurs on the first call.

**Detection principle:**
```cpp
unsigned int number = 1;
unsigned char* p = (unsigned char*)&number;
// if p[0] == 0 -> big endian (MSB first)
// if p[0] == 1 -> little endian (LSB first)
```

How integer `1` is laid out in memory:
- Little-endian: `[0x01, 0x00, 0x00, 0x00]` -- least significant byte first
- Big-endian: `[0x00, 0x00, 0x00, 0x01]` -- most significant byte first

### `host_has_little_endianness()`

```cpp
inline bool host_has_little_endianness(void);
```

Returns `true` if the host is little-endian. Also uses a static variable for caching.

### `has_host_endianness(endianness)`

```cpp
inline bool has_host_endianness(tlm_endianness endianness);
```

Checks whether the given endianness matches the host's endianness. Commonly used to decide whether endian conversion is needed:

```cpp
if (!has_host_endianness(target_endianness)) {
  // need endian conversion
  tlm_to_hostendian_generic<uint32_t>(&txn, bus_width);
}
```

## Source Location

`ref/systemc/src/tlm_core/tlm_2/tlm_generic_payload/tlm_helpers.h`

## Related Files

- [tlm_endian_conv.md](tlm_endian_conv.md) - Endian conversion functions
- [tlm_generic_payload.md](tlm_generic_payload.md) - Generic payload
