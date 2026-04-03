# display -- Result Display Module

> **File**: `display.h`, `display.cpp` | **Role**: Data sink

## Software Analogy

`display` is the endpoint of the pipeline, responsible for printing the final result. It is like:

- The stdout of the last command in a Unix pipe
- A log sink / console appender
- The Load step of an ETL pipeline (writing the result somewhere)

```python
# Python analogy
def display(value):
    print(f"result = {value}")
```

## Module Structure

### Header (display.h)

```cpp
SC_MODULE(display) {
    sc_in<bool>   clk;
    sc_in<double> in;     // from stage3.powr

    void entry();

    SC_CTOR(display) {
        SC_METHOD(entry);
        dont_initialize();
        sensitive << clk.pos();
    }
};
```

### Implementation (display.cpp)

```cpp
void display::entry() {
    printf("result = %g\n", in.read());
}
```

## Key Concepts

### The Simplest Module

`display` is the simplest module in the entire pipeline. It has only one input port, no output ports, and does just one thing: read the input and print it.

This module demonstrates an important SystemC design philosophy: **even the simplest functionality is wrapped in an independent module**. In software engineering, this is equivalent to the single responsibility principle -- each module is responsible for only one thing.

### printf vs cout

This example uses C-style `printf` rather than C++ `cout`. Both can be used in SystemC, but `printf` has advantages in certain situations:

- Format control is more intuitive (`%g` automatically selects the best floating-point display format)
- No need to include `<iostream>`
- In multi-process environments, `printf` is typically atomic (prints an entire line at once)

The `%g` format specifier automatically chooses the shorter format between `%f` (fixed-point) and `%e` (scientific notation), suitable for displaying floating-point numbers with uncertain ranges.

### The Role of a Sink Module

In hardware design, a sink module is usually not just `printf`. In a real system, it could be:

- **DAC (Digital-to-Analog Converter)**: converts digital signals to voltage output
- **UART TX**: sends data through a serial port
- **Memory controller**: writes results to memory

In SystemC simulation, `printf` is the most common means of debugging and verification, equivalent to `console.log` in software.
