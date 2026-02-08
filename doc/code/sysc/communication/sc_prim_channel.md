# SystemC Primitive Channel Analysis

> **File**: `ref/systemc/src/sysc/communication/sc_prim_channel.cpp`

## 1. Overview
`sc_prim_channel` is the base class for all primitive channels (channels that support the request-update evaluation phase, like `sc_signal`).

## 2. Update Mechanism
Primitive channels need to separate the "request" (write) from the "update" (commit) to ensure determinism (delta cycles).

### 2.1 Registry (`sc_prim_channel_registry`)
- Tracks all primitive channels.
- **`perform_update()`**: Called by the kernel at the end of the delta cycle. It iterates through the list of channels that requested an update and calls their `update()` method.

### 2.2 Async Updates
- Supports thread-safe updates from external threads (e.g., `async_request_update`).
- Uses a mutex-protected list (`async_update_list`) to safely queue updates which are then processed by the kernel in the main thread.

## 3. Key Takeaways
1.  **determinism**: The `request_update()` -> `update()` two-phase commit is the heart of SystemC's simulation semantics for signals.
2.  **Thread Safety**: The file contains specific logic (`sc_scoped_lock`) to handle external inputs safely.
