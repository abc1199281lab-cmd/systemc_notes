# scfx_string.h -- Internal String Class

## Overview

`scfx_string` is a **simple dynamic string class** designed specifically for fixed-point string conversion. Instead of using `std::string`, it manages its own memory buffer and provides character-by-character append and index operations.

## Everyday Analogy

`scfx_string` is like a "tear-off notepad." You can write characters one by one. When the page is full, it automatically switches to a larger page. You can also tear off (discard) a few characters from the end, or remove a character from the middle.

## Class Details

### Member Variables

| Member | Type | Description |
|--------|------|-------------|
| `m_len` | `size_t` | Current string length |
| `m_alloc` | `size_t` | Allocated buffer size |
| `m_buffer` | `char*` | Character buffer |

### Main Methods

| Method | Description |
|--------|-------------|
| `scfx_string()` | Construct with initial size = `BUFSIZ` |
| `length()` | Get current length |
| `clear()` | Clear (reset length to zero, does not free memory) |
| `operator[](i)` | Access the i-th character (auto-expands) |
| `operator+=(char)` | Append a single character |
| `operator+=(const char*)` | Append a C string |
| `append(n)` | Increase length by n (assumes already filled via []) |
| `discard(n)` | Decrease length by n (truncate tail) |
| `remove(i)` | Remove the i-th character, shifting subsequent characters forward |
| `operator const char*()` | Implicit conversion to C string |

### Auto-Expansion

When accessing a position beyond the current allocated size via `operator[]`, `resize()` is automatically called, doubling the buffer size until it is sufficient:

```cpp
void resize(size_t i) {
    do {
        m_alloc *= 2;
    } while (i >= m_alloc);
    // allocate new buffer, copy old content
}
```

## Why Not Use std::string?

1. **Performance**: Fixed-point string conversion is a high-frequency operation; a simple custom implementation avoids the overhead of `std::string`'s reference counting and small string optimization
2. **Historical reasons**: `scfx_string` predates the standardization of `std::string` in C++
3. **Interface requirements**: Operations like `append(n)` and `discard(n)` have no direct equivalents in `std::string`

## Related Files

- `scfx_utils.h` -- Uses `scfx_string` for formatted output
- `scfx_rep.h` / `scfx_rep.cpp` -- `to_string()` method uses `scfx_string`
