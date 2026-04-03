# sc_pvector - Pointer Vector

## Overview

`sc_pvector` is a simple pointer vector wrapper based on `std::vector`, providing some additional convenience operations such as sorting and automatic expansion. It is a legacy container used internally by SystemC, and is a completely different class from the IEEE 1666 standard's `sc_vector`.

**Source file**: `sysc/utils/sc_pvector.h` (header only)

## Analogy

Imagine an expandable business card holder:
- You can insert new cards at the end (`push_back`)
- You can directly flip to the Nth card (`operator[]`)
- If you flip to card 100 but the holder only has 50 slots, it automatically expands to 101 slots
- You can sort the cards (`sort`)
- You can clear everything and start over (`erase_all`)

## Class Interface

```cpp
template<class T>
class sc_pvector {
public:
    typedef const T* const_iterator;
    typedef T* iterator;

    sc_pvector();
    sc_pvector(const sc_pvector<T>& rhs);
    ~sc_pvector();

    std::size_t size() const;
    iterator begin();
    iterator end();
    const_iterator begin() const;
    const_iterator end() const;

    T& operator[](unsigned int i);       // auto-expand
    const T& operator[](unsigned int i) const;

    T& fetch(int i);                     // no expansion
    const T& fetch(int i) const;

    T* raw_data();                       // get underlying array pointer
    const T* raw_data() const;

    void push_back(T item);
    void erase_all();
    void sort(CFT compar);              // C-style sort

    void put(T item, int i);            // directly set the ith element
    void decr_count();                  // remove the last element
    void decr_count(int k);             // remove the last k elements

protected:
    mutable std::vector<T> m_vector;
};
```

## Design Features

### Auto-expanding operator[]

```cpp
T& operator[](unsigned int i) {
    if (i >= m_vector.size()) m_vector.resize(i+1);
    return m_vector[i];
}
```

Unlike the standard `std::vector::operator[]`, `sc_pvector` automatically expands to the required size. This avoids out-of-bounds errors but may also hide logic bugs.

### C-style Sort

```cpp
typedef int (*CFT)(const void*, const void*);
void sort(CFT compar) {
    qsort(&m_vector[0], m_vector.size(), sizeof(void*), compar);
}
```

Uses the C standard library's `qsort`, since this class is mainly used for sorting pointer types.

### mutable Internal Vector

```cpp
mutable std::vector<T> m_vector;
```

Declared as `mutable` because the `const` version of `operator[]` may also trigger `resize` (auto-expand semantics).

### Iterators are Raw Pointers

```cpp
typedef const T* const_iterator;
typedef T* iterator;
```

Iterators directly use raw pointers to the underlying `std::vector` data, since `std::vector` guarantees contiguous memory allocation.

## Differences from sc_vector

| Feature | `sc_pvector` | `sc_vector` |
|---------|-------------|-------------|
| Standard support | Internal use | IEEE 1666 standard |
| Inherits `sc_object` | No | Yes |
| Automatic naming | No | Yes |
| Batch binding | No | Yes |
| Type | General-purpose | Must be `sc_object` subclass |

## Related Files

- [sc_vector.md](sc_vector.md) -- IEEE 1666 standard named object vector
- [sc_list.md](sc_list.md) -- Linked list container
