# stage3 -- Power Stage

> **File**: `stage3.h`, `stage3.cpp` | **Role**: Third pipeline stage

## Software Analogy

`stage3` is the last computation stage of the pipeline, calculating `prod` raised to the power of `quot`. This is the most computationally expensive step in the entire pipeline.

```python
# Python analogy
def stage3_transform(prod, quot):
    if prod > 0 and quot > 0:
        return prod ** quot
    else:
        return 0.0
```

## Module Structure

### Header (stage3.h)

```cpp
SC_MODULE(stage3) {
    sc_in<bool>    clk;
    sc_in<double>  prod;   // from stage2
    sc_in<double>  quot;   // from stage2
    sc_out<double> powr;   // prod ^ quot

    void entry();

    SC_CTOR(stage3) {
        SC_METHOD(entry);
        dont_initialize();
        sensitive << clk.pos();
    }
};
```

### Implementation (stage3.cpp)

```cpp
void stage3::entry() {
    double a = prod.read();
    double b = quot.read();

    if (a > 0.0 && b > 0.0)
        powr.write(pow(a, b));
    else
        powr.write(0.0);      // negative/zero value guard
}
```

## Key Concepts

### Why Is the Positive Value Guard Needed?

The `pow(base, exponent)` function produces problems with certain inputs:

| base | exponent | `pow()` result | Problem |
|---|---|---|---|
| Positive | Positive | Normal | None |
| 0 | Positive | 0 | Usually acceptable |
| Negative | Non-integer | NaN | Mathematically undefined (complex number) |
| Positive | Negative | Normal (reciprocal) | May overflow |
| 0 | 0 | 1 (by convention) | Mathematically debatable |

The code takes the most conservative approach: it only computes when **base > 0 and exponent > 0**; otherwise, it outputs `0.0`.

This follows the same philosophy as stage2's division-by-zero guard -- in hardware design, it is better to output a safe default value than to let `NaN` or `Inf` corrupt downstream data.

### Pipeline Latency Accumulation

By stage3, the data has passed through three pipeline stages. Since each stage reads and outputs on the positive edge of the clock, and `sc_signal` value updates have a one delta cycle delay, the data stage3 sees was actually produced by numgen **several clocks earlier**.

```
Clock 1: numgen writes (a1, b1)
Clock 2: stage1 reads (a1, b1), writes (sum1, diff1)
          numgen writes (a2, b2)
Clock 3: stage2 reads (sum1, diff1), writes (prod1, quot1)
          stage1 reads (a2, b2), writes (sum2, diff2)
          numgen writes (a3, b3)
Clock 4: stage3 reads (prod1, quot1), writes (powr1)
          ...(and so on)
```

This is the core concept of pipelining: **latency increases** (the first result takes 4 clocks to appear), but **throughput is unaffected** (after that, one result comes out every clock).

## Computation Example

Continuing from the previous values (note the pipeline delay):

- Input: `prod = 8455.30`, `quot = 6.41`
- Both are > 0, so compute `pow(8455.30, 6.41)`
- The result is a very large number

Since `prod` is generally large and `quot` is not small either, the result of `pow()` tends to be very large or even overflow. This is expected behavior in the example -- this is a teaching example, not a practical numerical computation system.
