# scfx_pow10.h / .cpp -- Power-of-10 Computation and Caching

## Overview

The `scfx_pow10` class computes and caches **powers of 10** in arbitrary precision representation. This is primarily used for decimal conversion between strings and fixed-point numbers (e.g., `"3.14"` -> internal representation).

## Everyday Analogy

Imagine you are doing decimal-to-binary conversion and repeatedly need values like 10, 100, 1000, 0.1, 0.01, etc. Recomputing them every time is wasteful, so you write these values in a "lookup table." `scfx_pow10` is this lookup table, and it computes values lazily on first access, then looks them up directly afterward.

## Class Details

```cpp
class scfx_pow10 {
public:
    scfx_pow10();
    ~scfx_pow10();

    scfx_rep operator() (int);  // returns 10^n as scfx_rep

private:
    scfx_rep* pos(int);  // compute positive power
    scfx_rep* neg(int);  // compute negative power

    scfx_rep m_pos[SCFX_POW10_TABLE_SIZE]; // 10^0, 10^1, ..., 10^31
    scfx_rep m_neg[SCFX_POW10_TABLE_SIZE]; // 10^0, 10^-1, ..., 10^-31
};
```

### Cache Table

| Array | Size | Contents |
|-------|------|----------|
| `m_pos[32]` | 32 | Arbitrary precision values for 10^0 through 10^31 |
| `m_neg[32]` | 32 | Arbitrary precision values for 10^0 through 10^-31 |

### `operator()` Usage

```cpp
scfx_pow10 pow10;
scfx_rep val = pow10(5);   // returns 10^5 = 100000
scfx_rep val2 = pow10(-3); // returns 10^-3 = 0.001
```

For exponents beyond the cache range (|n| > 31), the result is computed by combining already-cached values.

## Why Are Arbitrary Precision Powers of 10 Needed?

Floating-point numbers cannot exactly represent decimal fractions like 0.1, 0.01, etc. (because they are infinite repeating fractions in binary). In fixed-point string conversion, precise decimal computation is needed, so `scfx_rep` must be used to represent powers of 10.

## Related Files

- `scfx_rep.h` -- `scfx_rep` class for storing arbitrary precision values
- `scfx_rep.cpp` -- Uses `scfx_pow10` in string parsing
