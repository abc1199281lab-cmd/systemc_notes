# sc_fxnum_observer.h / .cpp -- Fixed-Point Number Observer

## Overview

`sc_fxnum_observer` and `sc_fxnum_fast_observer` are abstract base classes for the Observer Pattern, allowing users to receive notifications when fixed-point objects are constructed, destructed, read, or written.

## Everyday Analogy

Like a bank account's "transaction notifications." Every time there is a deposit or withdrawal (write), or someone checks the balance (read), the bank sends you a notification. `sc_fxnum_observer` is this notification system.

## Class Details

### `sc_fxnum_observer` -- Arbitrary Precision Version

```cpp
class sc_fxnum_observer {
protected:
    sc_fxnum_observer() {}
    virtual ~sc_fxnum_observer() {}

public:
    virtual void construct( const sc_fxnum& );  // called on creation
    virtual void destruct( const sc_fxnum& );   // called on destruction
    virtual void read( const sc_fxnum& );       // called on read access
    virtual void write( const sc_fxnum& );      // called on write access

    static sc_fxnum_observer* (*default_observer)();  // factory function pointer
};
```

### `sc_fxnum_fast_observer` -- Limited Precision Version

Structurally identical to `sc_fxnum_observer`, but operates on `sc_fxnum_fast`.

### Default Observer

`default_observer` is a static function pointer, defaulting to `NULL` (no observer). Users can set a factory function to automatically install an observer for all newly created fixed-point objects.

## Conditional Compilation

Observer functionality is controlled by the `SC_ENABLE_OBSERVERS` macro:

- **When enabled**: Observer macros expand to actual notification calls
- **When disabled**: Observer macros expand to no-ops (zero overhead)

```cpp
#ifdef SC_ENABLE_OBSERVERS
#define SC_FXNUM_OBSERVER_READ_(object) \
    SC_OBSERVER_(object, sc_fxnum_observer*, read)
#else
#define SC_FXNUM_OBSERVER_READ_(object)  // empty
#endif
```

## Observer Macros

| Macro | Trigger |
|-------|---------|
| `SC_FXNUM_OBSERVER_CONSTRUCT_` | Fixed-point object construction complete |
| `SC_FXNUM_OBSERVER_DESTRUCT_` | Fixed-point object about to be destructed |
| `SC_FXNUM_OBSERVER_READ_` | Fixed-point value read |
| `SC_FXNUM_OBSERVER_WRITE_` | Fixed-point value written |

The fast version has corresponding `SC_FXNUM_FAST_OBSERVER_*` macros.

## .cpp File

`sc_fxnum_observer.cpp` does only one thing: initialize the two static function pointers to `0`:

```cpp
sc_fxnum_observer* (*sc_fxnum_observer::default_observer)() = 0;
sc_fxnum_fast_observer* (*sc_fxnum_fast_observer::default_observer)() = 0;
```

## Use Cases

1. **Debugging**: Track when fixed-point variables are read or written
2. **Statistics**: Collect statistical information about fixed-point operations (how many quantizations, how many overflows)
3. **Verification**: Confirm that specific values are not accidentally modified

## Related Files

- `sc_fxdefs.h` -- Generic observer macros `SC_OBSERVER_`, `SC_OBSERVER_DEFAULT_`
- `sc_fxnum.h` -- Uses observer macros
- `sc_fxval_observer.h` -- Similar observer for `sc_fxval`
