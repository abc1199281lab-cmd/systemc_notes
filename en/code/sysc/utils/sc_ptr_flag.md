# sc_ptr_flag - Pointer Flag

## Overview

`sc_ptr_flag` is a template utility class that leverages pointer alignment properties to store a boolean flag in the least significant bit (LSB) of a pointer value. This is a memory-saving technique that eliminates the need for a separate boolean variable.

**Source file**: `sysc/utils/sc_ptr_flag.h` (header only)

## Analogy

Imagine every building's address is an even number (2, 4, 6...). Since the last digit is always even, that digit is "wasted". A clever mail carrier can use this spare digit to mark: "this building has registered mail to deliver".

Similarly: since objects in memory are always aligned (at least 2-byte aligned), the least significant bit of a pointer value is always 0. `sc_ptr_flag` uses this spare bit to store a boolean flag.

## Principle

```
Assume T has an alignment requirement of 4 bytes:
Normal pointer value:  ...1100 0100 0000  (last 2 bits are always 00)
Available space:       ─────────────┐
                                    └── This bit can be borrowed to store a flag!

With flag stored:      ...1100 0100 0001  (LSB = flag true)
Extracting pointer:    AND 11...1110      (mask out LSB to restore)
```

## Class Interface

```cpp
template<typename T>
class sc_ptr_flag {
public:
    typedef T* pointer;
    typedef T& reference;

    sc_ptr_flag();                          // default constructor
    sc_ptr_flag(pointer p, bool f = false); // pointer + flag
    sc_ptr_flag& operator=(pointer p);      // set pointer

    operator pointer() const;     // implicit conversion to pointer
    pointer operator->() const;   // arrow operator
    reference operator*() const;  // dereference operator

    pointer get() const;          // get pointer (without flag)
    void reset(pointer p);        // reset pointer (preserve flag)
    bool get_flag() const;        // get flag
    void set_flag(bool f);        // set flag

private:
    uintptr_type m_data;          // actual storage: pointer + flag
};
```

## Implementation Details

### Static Assertion

```cpp
static_assert(alignof(T) > 1,
    "Unsupported platform/type, need spare LSB of pointer to store flag");
```

Ensures that type `T` has an alignment requirement of at least 2; otherwise the least significant bit is not spare and this technique cannot be used.

### Bit Masks

```cpp
static const uintptr_type flag_mask    = 0x1;          // least significant bit
static const uintptr_type pointer_mask = ~flag_mask;   // all other bits
```

### Core Operations

```cpp
// Get pointer: mask out LSB
pointer get() const {
    return reinterpret_cast<pointer>(m_data & pointer_mask);
}

// Set pointer: preserve flag
void reset(pointer p) {
    m_data = reinterpret_cast<uintptr_type>(p)
           | static_cast<uintptr_type>(get_flag());
}

// Get flag: extract LSB
bool get_flag() const {
    return (m_data & flag_mask) != 0x0;
}

// Set flag: only modify LSB
void set_flag(bool f) {
    m_data = (m_data & pointer_mask)
           | static_cast<uintptr_type>(f);
}
```

## Use Cases

In the SystemC core, `sc_ptr_flag` is used wherever both a pointer and a boolean attribute need to be stored together, for example:
- Marking whether an object has already been processed
- Marking whether a link is in a particular state

The benefit is saving the space of a `bool` member variable (considering alignment, a `bool` can take up 8 bytes).

## Memory Savings Demonstration

```
Without sc_ptr_flag:
struct Node {
    Object* ptr;    // 8 bytes
    bool    flag;   // 1 byte + 7 bytes padding = 8 bytes
};                  // total 16 bytes

With sc_ptr_flag:
struct Node {
    sc_ptr_flag<Object> pf;  // 8 bytes (pointer + flag combined)
};                           // total 8 bytes
```

## Related Files

- [sc_machine.md](sc_machine.md) -- Platform detection (alignment requirements vary by platform)
