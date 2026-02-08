# SystemC Module Analysis

> **Files**:
> - `ref/systemc/src/sysc/kernel/sc_module.cpp`
> - `ref/systemc/src/sysc/kernel/sc_module_name.cpp`

## 1. Overview
`sc_module` is the container class for processes, ports, channels, and other modules. It is the fundamental building block of structural hierarchy.

## 2. The `sc_module` Lifecycle

### 2.1 Construction & `sc_module_name`
The construction of a module is a delicate dance to ensure hierarchy is captured correctly:
1.  **User**: `SC_MODULE(my_mod) { ... }` expands to a constructor taking `sc_module_name`.
2.  **`sc_module_name` Constructor**:
    - Pushes itself onto a stack in `sc_object_manager`.
    - This "marks" the start of a module's construction.
3.  **`sc_module` Constructor**:
    - Peeks at the `sc_module_name` stack to get its name.
    - Pushes *itself* as the **active object** in `sc_simcontext`.
    - Now, any child objects (ports, submodules) created will see this module as their parent.
4.  **`sc_module_name` Destructor**:
    - Calls `m_module_p->end_module()`.
    - This triggers `simcontext->hierarchy_pop()`, closing the scope.

### 2.2 `end_module()`
Explicitly closes the module's hierarchy scope. In modern SystemC (`SC_CTOR`), this is handled automatically by the `sc_module_name` RAII pattern.

## 3. Process Declaration
`sc_module` provides the backend for `SC_METHOD`, `SC_THREAD`, `SC_CTHREAD`.
- `declare_method_process`
- `declare_thread_process`
- `declare_cthread_process`
These functions create the process handle, register it with the kernel, and set up initial sensitivity.

## 4. Port Binding (`positional_bind`)
While named binding (`port(channel)`) is preferred, `sc_module` implements positional binding (deprecated `<<` operator or `operator()`).
- It iterates through the `m_port_vec` (registered ports) and binds them in order.

## 5. Hardware / RTL Mapping
| SystemC (`sc_module`) | Verilog / SystemVerilog |
| :--- | :--- |
| `sc_module` | `module ... endmodule` |
| `SC_CTOR` | `initial begin ... end` + instantiation logic |
| `sc_in`, `sc_out` members | Input/Output ports |
| Submodule members | Module instances |

## 6. Key Takeaways
1.  **RAII Hierarchy**: The use of `sc_module_name` is effectively a trick to detect the *start* and *end* of the constructor execution without requiring explicit `end_module()` calls from the user.
2.  **Process Container**: Modules are the "Hosts" (`sc_object_host`) for processes. When a process runs, its `m_parent` is the module it resides in.
3.  **Lifecycle Callbacks**: `before_end_of_elaboration`, `end_of_elaboration`, `start_of_simulation` are virtual methods hooked here to allow users to run code at specific phases.
