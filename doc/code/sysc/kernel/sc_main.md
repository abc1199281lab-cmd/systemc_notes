# SystemC Entry Point Analysis

> **Files**:
> - `ref/systemc/src/sysc/kernel/sc_main.cpp`
> - `ref/systemc/src/sysc/kernel/sc_main_main.cpp`

## 1. Overview
The entry point of a SystemC application is unique. Unlike standard C++ applications where `main()` is the user-defined entry point, SystemC libraries define `main()` themselves. The user is required to define `sc_main()` instead.

These two files act as a "wrapper" or "shim" to bridge the standard C++ `main()` to the SystemC kernel's initialization and finally to the user's `sc_main()`.

---

## 2. Code Analysis

### 2.1 `sc_main.cpp` - The True Entry Point
This file contains the actual C++ `main` function.

```cpp
// sysc/kernel/sc_main.cpp
int main( int argc, char* argv[] )
{
    // Forwards execution to sc_elab_and_sim defined in sc_main_main.cpp
    return sc_core::sc_elab_and_sim( argc, argv );
}
```

- **Role**: It simply captures the command line arguments and immediately delegates control to `sc_elab_and_sim`.
- **Design Pattern**: **Proxy/Wrapper**. It intercepts the program start to ensure SystemC initializes correctly before user code runs.

### 2.2 `sc_main_main.cpp` - The Execution Orchestrator
This file contains the core logic for setting up the SystemC environment.

#### Key Function: `sc_elab_and_sim`

```cpp
// sysc/kernel/sc_main_main.cpp
int sc_elab_and_sim( int argc, char* argv[] )
{
    // ... (Argument handling) ...

    try
    {
        pln(); // Print library name/copyright

        // 1. Initialization Marker
        sc_in_action = true; // Global flag indicating Simulator is active

        // 2. Call User's sc_main
        // Note: argv is copied to prevent modification by user code affecting internal state
        status = sc_main( argc, &argv_call[0] );

        // 3. Cleanup
        sc_in_action = false;
    }
    catch( ... )
    {
        // ... (Exception handling via sc_report_handler) ...
    }

    // ... (Deprecation warnings check) ...

    return status;
}
```

- **Safety Mechanism**: It wraps `sc_main` in a `try-catch` block to ensure that exceptions (like `sc_report`) are caught and handled gracefully by the SystemC report handler, rather than crashing the program.
- **State Management**: Uses `sc_in_action` simple global flag to tracking if the specific shim is running.
- **Argv Protection**: It makes a deep copy of `argv` before passing it to `sc_main`. This is defensive programming to protect the original command-line arguments.

---

## 3. Hardware / RTL Mapping
While this is pure software scaffolding, it parallels the **Testbench Top** in HDL.

| SystemC | Verilog / SystemVerilog |
| :--- | :--- |
| `main()` (Library defined) | Simulator Kernel (e.g., `vsim`, `irun`) |
| `sc_main()` (User defined) | `module top;` or `program automatic test;` |
| `sc_elab_and_sim` | The process of compiling and loading the design (`elaboration`) |

- **Concept**: Just as you don't write the "boot code" of a Verilog simulator, you don't write `main()` in SystemC. You write the top-level testbench (`sc_main`), and the kernel handles the boot sequence.

---

## 4. Key Takeaways based on Code
1.  **Don't define `main()`**: You will get a linker error (multiple definitions of `main`) if you try to define your own `main` function in a SystemC application.
2.  **Exception Handling**: The kernel is designed to catch exceptions thrown from your `sc_main`.
3.  **Library initialization**: `sc_elab_and_sim` is technically the "main loop" wrapper, although the actual simulation loop happens inside `sc_start()` which is called *within* your `sc_main()`.
