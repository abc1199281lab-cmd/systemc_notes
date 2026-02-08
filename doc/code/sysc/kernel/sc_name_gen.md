# SystemC Kernel: sc_name_gen

> **Source**: `ref/systemc/src/sysc/kernel/sc_name_gen.cpp`

## 1. Overview
`sc_name_gen` is the utility responsible for ensuring all SystemC objects have unique names. If a user provides a duplicate name (or no name), this class generates a unique suffix.

## 2. Mechanism

### 2.1 The Map
It maintains a hash map `m_unique_name_map`:
-   **Key**: `const char*` (The base name provided by the user).
-   **Value**: `int*` (A counter for how many times this name has been seen).

### 2.2 Generation Logic (`gen_unique_name`)
When requested to generate a unique name for `basename`:
1.  **First Occurrence**:
    -   If `preserve_first` is true (default for signals/modules): The returned name is exactly `basename`.
    -   Counter is initialized to 0.
2.  **Subsequent Occurrences** (Collision):
    -   The counter is incremented.
    -   The new name becomes `basename_N` (e.g., `my_signal_0`, `my_signal_1`).

## 3. Usage
-   Used by `sc_object_manager` when creating instances.
-   Ensures that even if the user does `sc_signal<int> sig("val");` in a loop, each signal gets a unique hierarchical name internally (though explicit naming is preferred).

## 4. Key Takeaways
-   **Uniqueness**: Guarantees unique names within the same scope (usually sibling objects).
-   **Determinism**: As long as object creation order is deterministic, the generated names will be deterministic.
