# sc_fxval_observer.h / .cpp -- Fixed-Point Value Observer

## Overview

`sc_fxval_observer` and `sc_fxval_fast_observer` provide abstract base classes for the Observer Pattern, used to monitor construction, destruction, read, and write events of `sc_fxval` and `sc_fxval_fast` objects.

## Everyday Analogy

Like a courier's "package tracking" system. Every time a package (value) is created, moved, or opened, the system records a tracking entry.

## Difference from `sc_fxnum_observer`

| Feature | `sc_fxval_observer` | `sc_fxnum_observer` |
|---------|---------------------|---------------------|
| Monitored object | `sc_fxval` (intermediate values) | `sc_fxnum` (fixed-point variables) |
| Use case | Track computation process | Track variable state |
| Activation | `SC_ENABLE_OBSERVERS` | `SC_ENABLE_OBSERVERS` |

## Class Details

### `sc_fxval_observer`

```cpp
class sc_fxval_observer {
protected:
    sc_fxval_observer() {}
    virtual ~sc_fxval_observer() {}

public:
    virtual void construct( const sc_fxval& );
    virtual void destruct( const sc_fxval& );
    virtual void read( const sc_fxval& );
    virtual void write( const sc_fxval& );

    static sc_fxval_observer* (*default_observer)();
};
```

All virtual methods have empty default implementations (no-op); users need to inherit and override them.

### Conditional Compilation Macros

```cpp
#ifdef SC_ENABLE_OBSERVERS
#define SC_FXVAL_OBSERVER_READ_(object) \
    SC_OBSERVER_(object, sc_fxval_observer*, read)
#else
#define SC_FXVAL_OBSERVER_READ_(object)  // empty
#endif
```

## .cpp File

`sc_fxval_observer.cpp` initializes the static factory function pointers to `0`:

```cpp
sc_fxval_observer* (*sc_fxval_observer::default_observer)() = 0;
sc_fxval_fast_observer* (*sc_fxval_fast_observer::default_observer)() = 0;
```

## Related Files

- `sc_fxval.h` -- Value class that uses observer macros
- `sc_fxnum_observer.h` -- Similar observer for `sc_fxnum`
- `sc_fxdefs.h` -- Generic observer macro definitions
