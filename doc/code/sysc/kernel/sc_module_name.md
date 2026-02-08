# SystemC Kernel: sc_module_name

> **Source**: `ref/systemc/src/sysc/kernel/sc_module_name.cpp`

## 1. Overview
`sc_module_name` is a clever helper class used in SystemC to transparently pass module names to the `sc_module` constructor. It solves the C++ limitation where base class constructors (`sc_module`) are called before derived class members.

## 2. The Problem
When defining a module:
```cpp
SC_MODULE(my_module) {
    SC_CTOR(my_module) { ... }
};
```
The macro expands to a constructor taking `sc_module_name`:
```cpp
typedef my_module SC_CURRENT_USER_MODULE;
my_module( ::sc_core::sc_module_name name ) : ::sc_core::sc_module(name)
```
SystemC needs to know the module's name *while* the `sc_module` base constructor is running, to register it properly in the hierarchy.

## 3. Implementation Mechanism

### 3.1 Constructor (`sc_module_name`)
When you instantiate a module: `my_module m("inst_name");`
1.  A temporary `sc_module_name` object is constructed with "inst_name".
2.  **Push**: It immediately calls `m_simc->get_object_manager()->push_module_name(this)`.
3.  This places the name on a stack in the `sc_object_manager`.

### 3.2 During `sc_module` Construction
The `sc_module` constructor calls `sc_module_name::operator const char*()` (implicitly or explicitly) or retrieves the top of the name stack to get the name.

### 3.3 Destructor (`~sc_module_name`)
After the `sc_module` constructor returns (and the derived module constructor body begins execution):
1.  The temporary `sc_module_name` object is destroyed.
2.  **Pop**: It calls `m_simc->get_object_manager()->pop_module_name()`.
3.  **End Module**: It calls `m_module_p->end_module()` if linked, which signifies that the module's construction phase is almost done (processes might be registered).

## 4. Key Takeaways
-   **Transient Object**: `sc_module_name` is designed to be short-lived, existing only during the construction of a module.
-   **Stack-Based**: Uses a global (per-simcontext) stack to pass names, enabling safe hierarchical construction.
-   **Safety**: Includes checks to ensure names are popped in the reverse order of pushing (LIFO).
