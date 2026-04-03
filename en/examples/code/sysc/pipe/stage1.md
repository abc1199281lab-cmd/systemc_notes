# stage1 -- Addition/Subtraction Stage

> **File**: `stage1.h`, `stage1.cpp` | **Role**: First pipeline stage

## Software Analogy

`stage1` is like the first **transform** step in a pipeline. It reads two input values and produces two output values (sum and diff).

```python
# Python analogy
def stage1_transform(a, b):
    return a + b, a - b
```

In a Unix pipe, it is equivalent to an `awk` command that reads input, performs computation, and outputs results.

## Module Structure

### Header (stage1.h)

```cpp
SC_MODULE(stage1) {
    sc_in<bool>    clk;    // clock input
    sc_in<double>  in1;    // input a (from numgen.out1)
    sc_in<double>  in2;    // input b (from numgen.out2)
    sc_out<double> sum;    // output a + b
    sc_out<double> diff;   // output a - b

    void entry();

    SC_CTOR(stage1) {
        SC_METHOD(entry);
        dont_initialize();
        sensitive << clk.pos();
    }
};
```

### Implementation (stage1.cpp)

```cpp
void stage1::entry() {
    double a = in1.read();   // read from input port
    double b = in2.read();
    sum.write(a + b);        // write to output port
    diff.write(a - b);
}
```

## Key Concepts

### sc_in and sc_out -- Module External Interfaces

`sc_in<T>` and `sc_out<T>` are SystemC port types that define a module's input and output interfaces.

Their software analogies:

| SystemC | Software Analogy |
|---|---|
| `sc_in<double>` | Function parameter (`const double& input`) |
| `sc_out<double>` | Function return value (`double& output`) |
| `sc_inout<double>` | Read-write reference (`double& ref`) |

### read() and write() Methods

Ports cannot be assigned directly with `=`; they must be accessed through `read()` and `write()` methods:

```cpp
// Correct
double a = in1.read();
sum.write(a + b);

// Wrong -- cannot access directly
// double a = in1;       // compile error
// sum = a + b;          // compile error
```

Why is it designed this way? Because `read()` and `write()` may involve delta cycle synchronization mechanisms behind the scenes. Direct assignment cannot guarantee simulation correctness.

This is similar to React's `useState` philosophy: you cannot modify state directly; you must update it through `setState()`, because the framework needs to track changes.

### Relationship Between Ports and Signals

Ports do not store data themselves; they must be **bound** to an `sc_signal` to function. The `sc_signal` is where the value is actually stored.

```
sc_out<double> sum  --bound to-->  sc_signal<double> sum_sig  <--bound to--  sc_in<double> sum
(stage1 output)                    (declared in main.cpp)                    (stage2 input)
```

This is like a Unix pipe: the `|` symbol creates an anonymous pipe connecting the stdout of the left process to the stdin of the right process.

## Computation Logic

Using the first clock as an example:

- Input: `a = 134.56`, `b = 98.24`
- Output: `sum = 232.80`, `diff = 36.32`

Note: due to the delta cycle delay in the pipeline, the values stage1 reads are actually the values numgen wrote in the **previous** delta cycle. This is a characteristic of `sc_signal` -- written values are not readable until the next delta cycle.
