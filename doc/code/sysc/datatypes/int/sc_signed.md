# SystemC Data Types: Arbitrary Precision Signed Integer (`sc_signed`)

> **Source**: `ref/systemc/src/sysc/datatypes/int/sc_signed.cpp`

## 1. Overview
`sc_signed` is the base for `sc_bigint<W>`. It supports signed integers of arbitrary width, limited only by memory.

## 2. Implementation

### 2.1 Storage (`sc_digit`)
-   Uses dynamic allocation: `sc_digit* digit`.
-   `sc_digit` is typically a 32-bit `unsigned int`.
-   Negative numbers are stored in **2's complement** representation across the digit array.

### 2.2 Arithmetic
-   Operations are implemented via function calls (e.g., `add_signed_friend`, `mult_signed_friend` in `sc_signed_ops.h/cpp`) rather than native operators.
-   This involves loop-based carry/borrow propagation across the digit array.

### 2.3 Double Conversion
Includes logic to convert `double` to `sc_signed`, handling edge cases and using `floor` and `fmod` to extract digits.

## 3. Concatenation and Part Selection
-   **`concat_get_data`**: Complex logic to extract bits from the digit array into a destination buffer, handling cases where the selection spans multiple digits, is not byte-aligned, or involves partial words.
-   **`check_if_outside`**: Bounds checking is critical here to prevent buffer overflows.

## 4. Key Takeaways
-   **Sign Extension**: `sc_signed` automatically handles sign extension when converting to larger types or performing arithmetic.
-   **Performance**: Slower than `sc_int` due to dynamic memory and software implementations of arithmetic. Use only when `W > 64`.
