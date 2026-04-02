# sc_int32_mask.cpp — 32-Bit Mask Lookup Table

## Overview

`sc_int32_mask.cpp` initializes the `mask_int` lookup table for 32-bit platforms. This table is used for part-selection operations of `sc_int` and `sc_uint`, providing precomputed bitmasks to avoid redundant calculations at runtime.

**Source file:**
- `ref/systemc/src/sysc/datatypes/int/sc_int32_mask.cpp`

**Note:** This file is only compiled on 32-bit platforms (`_32BIT_`). 64-bit platforms use [sc_int64_mask.cpp](sc_int64_mask.md).

## Everyday Analogy

Imagine you are a tailor who frequently needs to cut fabric into various sizes. Instead of measuring with a ruler every time, you make a set of "cutting templates" — one for 1 cm, one for 2 cm, one for 3 cm... When you need a particular size, just grab the template.

`mask_int` is this set of "cutting templates," with mask values precomputed for every possible bit range `[i:j]`.

## Data Structure

```cpp
const uint_type mask_int[SC_INTWIDTH][SC_INTWIDTH];
```

On 32-bit platforms, `SC_INTWIDTH` = 32, so this is a 32x32 two-dimensional array.

`mask_int[i][j]` contains a mask value that sets all bits outside the range from bit `j` to bit `i` to 1. This is used to extract or modify a specified range of bits.

### Examples

```
mask_int[0][0] = 0xFFFFFFFE  // bit 0 cleared
mask_int[1][0] = 0xFFFFFFFC  // bits 1:0 cleared
mask_int[1][1] = 0xFFFFFFFD  // bit 1 cleared
mask_int[2][0] = 0xFFFFFFF8  // bits 2:0 cleared
```

## Design Rationale

### Lookup Table vs. On-the-Fly Calculation

The lookup table approach sacrifices a small amount of memory (32x32 = 1024 32-bit integers = 4KB) in exchange for O(1) time complexity for part-selection operations. This is well worth it in hardware simulation, as bit operations are among the most frequent operations.

## Related Files

- [sc_int64_mask.md](sc_int64_mask.md) — 64-bit version
- [sc_int_base.md](sc_int_base.md) — Class that uses this mask table
- [sc_uint_base.md](sc_uint_base.md) — Class that uses this mask table
