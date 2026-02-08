# SystemC Spawn Options Analysis

> **File**: `ref/systemc/src/sysc/kernel/sc_spawn_options.cpp`

## 1. Overview
Provides the implementation for `sc_spawn_options`, which allows users to configure dynamic processes created via `sc_spawn`.

## 2. Mechanism
It primarily acts as a container for settings that are applied when the process is created. The most complex part implemented here is the **Reset Signal Registration**.

### 2.1 Reset Registration (`sc_spawn_reset`)
- Since `sc_spawn` creates processes dynamically, we can't use the static `reset_signal_is` easily during elaboration.
- `sc_spawn_options` allows users to specify reset signals (`async_reset_signal_is`, `reset_signal_is`).
- These are stored in a vector `m_resets`.
- **`specify_resets()`**: Called by the kernel typically after the process is created but before it runs, to actually register the new process with the `sc_reset` system.

## 3. Key Takeaways
1.  **Dynamic Configuration**: Allows dynamic processes to have almost full feature parity with static processes (sensitive lists, dont_initialize, stack size, and resets).
