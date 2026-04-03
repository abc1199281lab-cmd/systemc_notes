# stage2 -- Multiplication/Division Stage

> **File**: `stage2.h`, `stage2.cpp` | **Role**: Second pipeline stage

## Software Analogy

`stage2` receives sum and diff from stage1, and computes their **product** and **quotient**. This is like the second transform step in an ETL pipeline.

```python
# Python analogy
def stage2_transform(sum_val, diff_val):
    prod = sum_val * diff_val
    quot = sum_val / diff_val if diff_val != 0 else 5.0
    return prod, quot
```

## Module Structure

### Header (stage2.h)

```cpp
SC_MODULE(stage2) {
    sc_in<bool>    clk;
    sc_in<double>  sum;    // from stage1
    sc_in<double>  diff;   // from stage1
    sc_out<double> prod;   // sum * diff
    sc_out<double> quot;   // sum / diff

    void entry();

    SC_CTOR(stage2) {
        SC_METHOD(entry);
        dont_initialize();
        sensitive << clk.pos();
    }
};
```

### Implementation (stage2.cpp)

```cpp
void stage2::entry() {
    double a = sum.read();
    double b = diff.read();

    prod.write(a * b);

    if (b != 0.0)
        quot.write(a / b);
    else
        quot.write(5.0);   // division-by-zero guard: use default value
}
```

## Key Concepts

### Division-by-Zero Guard

`stage2` checks whether `diff` is zero before performing division. If it is zero, a safe default value of `5.0` is output to avoid producing `Inf` or `NaN`.

This is a common **defensive programming** pattern:

```python
# Python -- similar defensive pattern
try:
    result = a / b
except ZeroDivisionError:
    result = default_value
```

```go
// Go -- similar defensive pattern
if b != 0 {
    quot = a / b
} else {
    quot = 5.0
}
```

In hardware design, defensive programming is especially important because:

1. **No exception mechanism**: hardware has no try-catch; all edge cases must be prevented in the logic.
2. **Undefined behavior is more dangerous**: software can crash and restart, but erroneous hardware signals can propagate through the entire system.
3. **Debugging is difficult**: tracing `NaN` propagation in hardware simulation is very painful; a single division by zero can corrupt all downstream computations.

### Why Is the Default Value 5.0?

The original code chooses `5.0` as the default value for division by zero. This is a relatively arbitrary choice. In a real project, you might:

- Output `0.0` (most conservative)
- Output the last valid value (hold last valid)
- Set an error flag to notify downstream

## Computation Logic

Using the first set of output from stage1 as an example:

- Input: `sum = 232.80`, `diff = 36.32`
- Output: `prod = 232.80 * 36.32 = 8,455.30`, `quot = 232.80 / 36.32 = 6.41`

Since `diff` will keep decrementing in this test data, the difference between `a` and `b` may eventually approach zero (if the simulation runs enough cycles), at which point the division-by-zero guard will activate.
