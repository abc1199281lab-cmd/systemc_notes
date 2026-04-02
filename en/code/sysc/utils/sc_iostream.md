# sc_iostream - Portable I/O Stream Header

## Overview

`sc_iostream.h` is a simple I/O stream header wrapper file that provides a unified include for commonly used C++ standard I/O headers. In the early days of SystemC, different compilers had varying levels of support for the C++ standard library, and this file served to abstract away those differences.

**Source file**: `sysc/utils/sc_iostream.h` (header only)

## Current Status

This file is currently marked as **deprecated**, since all modern C++ compilers properly support the standard I/O headers. It now simply includes the following headers:

```cpp
#include <iostream>
#include <sstream>
#include <fstream>
#include <cstddef>
#include <cstring>
```

## Analogy

Imagine power outlet adapters for different countries: in the early days, outlet standards varied by country, so you needed a universal adapter. Now most countries support USB-C, making the adapter no longer necessary, but it is kept around for compatibility with older devices.

`sc_iostream.h` is such an "adapter" -- it was once very important, but now exists only to avoid breaking legacy code.

## Related Files

- [sc_string.md](sc_string.md) -- String utilities that use this header
