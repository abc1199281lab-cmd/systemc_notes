# numgen -- Number Generator

> **File**: `numgen.h`, `numgen.cpp` | **Role**: Data source

## Software Analogy

`numgen` is like a **sensor** or **data generator** that produces a new set of readings each clock tick.

In the software world, this is similar to:
- A Python generator function (yielding a set of values each time)
- A Python asyncio StreamReader (reading one data item at a time)
- A Kafka producer (sending an event at each interval)

```python
# Python analogy
def numgen():
    a, b = 134.56, 98.24
    while True:
        yield a, b
        a -= 1.5
        b -= 2.8
```

## Module Structure

### Header (numgen.h)

```cpp
SC_MODULE(numgen) {
    sc_in<bool>   clk;    // clock input
    sc_out<double> out1;   // output a
    sc_out<double> out2;   // output b

    void entry();          // called on each clock positive edge

    SC_CTOR(numgen) {
        SC_METHOD(entry);           // register as SC_METHOD
        dont_initialize();          // do not call automatically at simulation start
        sensitive << clk.pos();     // trigger only on clock positive edge
    }
};
```

### Implementation (numgen.cpp)

```cpp
void numgen::entry() {
    out1.write(a);   // output current a
    out2.write(b);   // output current b
    a -= 1.5;        // decrement a
    b -= 2.8;        // decrement b
}
```

## Key Concepts

### How SC_METHOD Works

`SC_METHOD(entry)` tells SystemC: "Each time the specified sensitivity event occurs, call the `entry()` function."

This is very similar to registering a callback in Python:

```python
# Python analogy
def on_posedge():
    output1 = a
    output2 = b
    a -= 1.5
    b -= 2.8

clock.on_posedge(on_posedge)
```

Important restriction: SC_METHOD **cannot** call `wait()`. It must complete all work and return within a single invocation. If you need to pause midway, use SC_THREAD instead.

### What dont_initialize() Does

By default, SystemC automatically calls all processes once at the start of simulation (time 0), regardless of whether the trigger condition is met. `dont_initialize()` prevents this behavior.

Why is it needed? Because `numgen` should only output data on clock positive edges. Without `dont_initialize()`, it would write initial values at time 0, which may not be the desired behavior.

### sensitive << clk.pos()

`sensitive` is the SystemC sensitivity list, which determines what events trigger this process. `clk.pos()` means "the positive edge (rising edge, from 0 to 1) of the clock signal."

This is equivalent to the hardware:

```
// Hardware analogy (Verilog)
always @(posedge clk) begin
    out1 <= a;
    out2 <= b;
end
```

### SC_METHOD vs SC_THREAD Comparison

| | SC_METHOD | SC_THREAD |
|---|---|---|
| Analogy | callback / event handler | coroutine / Python coroutine (asyncio) |
| Execution | Runs from start to finish each trigger | Can pause with `wait()` |
| `wait()` | **Forbidden** | Allowed |
| State | Saved in member variables (e.g., `a`, `b`) | Via local variables + `wait()` |
| Performance | Fast (no stack switch) | Slower (requires saving/restoring stack) |

In this example, `numgen`'s logic is simple (write and decrement), so SC_METHOD is sufficient. If you need a complex multi-step flow (e.g., send a header, wait for a response, then send the payload), SC_THREAD would be more appropriate.

## Output Data Sequence

| Clock | out1 (a) | out2 (b) |
|---|---|---|
| 1 | 134.56 | 98.24 |
| 2 | 133.06 | 95.44 |
| 3 | 131.56 | 92.64 |
| 4 | 130.06 | 89.84 |
| ... | -1.5/cycle | -2.8/cycle |
