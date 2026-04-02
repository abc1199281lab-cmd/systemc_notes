# sc_int64_mask.cpp — 64-Bit Mask Lookup Table

## Overview

`sc_int64_mask.cpp` initializes the `mask_int` lookup table for 64-bit platforms. It has the same functionality as `sc_int32_mask.cpp`, but the mask values are 64-bit and the array size is 64x64.

**Source file:**
- `ref/systemc/src/sysc/datatypes/int/sc_int64_mask.cpp`

## Differences from the 32-Bit Version

| Feature | sc_int32_mask | sc_int64_mask |
|---------|---------------|---------------|
| Conditional compilation | `#ifdef _32BIT_` | Unconditional (default) |
| `SC_INTWIDTH` | 32 | 64 |
| Array size | 32x32 | 64x64 |
| Element type | `uint32_t` | `uint64_t` |
| Memory usage | ~4 KB | ~32 KB |

## Data Structure

```cpp
const uint_type mask_int[SC_INTWIDTH][SC_INTWIDTH];
// SC_INTWIDTH = 64 on 64-bit platforms
```

### Example Values

```
mask_int[0][0] = 0xFFFFFFFFFFFFFFFE  // bit 0 cleared
mask_int[1][0] = 0xFFFFFFFFFFFFFFFC  // bits 1:0 cleared
mask_int[1][1] = 0xFFFFFFFFFFFFFFFD  // bit 1 cleared
```

## Design Rationale

Modern systems are almost all 64-bit, so this file is the version actually used. Trading 32 KB of memory for high-speed bit operations is a very reasonable trade-off in hardware simulation scenarios.

## Related Files

- [sc_int32_mask.md](sc_int32_mask.md) — 32-bit version
- [sc_int_base.md](sc_int_base.md) — Uses this mask table
- [sc_uint_base.md](sc_uint_base.md) — Uses this mask table
