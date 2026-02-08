# TLM Simple Target Socket Analysis

> **File**: `ref/systemc/src/tlm_utils/simple_target_socket.h`

## 1. Overview
The `simple_target_socket` is a powerful utility in `tlm_utils` that simplifies target implementation and provides automatic transport adaptation.

## 2. Dynamic Adaptation (The "Magic")
This socket can adapt between Blocking and Non-Blocking calls.
- **Blocking Target, Non-Blocking Initiator**: If the target only registers `b_transport` but the initiator calls `nb_transport_fw`, the socket's internal `fw_process`:
    1.  Catches the `nb` call.
    2.  Spawns a dynamic `SC_THREAD` ( `nb2b_thread`).
    3.  Calls the user's `b_transport` in that thread.
    4.  Handles the return path (sending `BEGIN_RESP/END_RESP`) when the blocking call returns.
- **Non-Blocking Target, Blocking Initiator**: If the target only registers `nb_transport` but the initiator calls `b_transport`, the socket:
    1.  Calls `nb_transport_fw`.
    2.  Waits on an internal event (`m_peq`) until the response is complete.

## 3. Callback Registration
Like the simple initiator socket, it allows registering callbacks instead of inheriting from `tlm_fw_transport_if`.
```cpp
socket.register_b_transport(this, &MyModule::my_b_transport);
```

## 4. Variants
- **`simple_target_socket<MODULE>`**: Standard version.
- **`simple_target_socket_tagged<MODULE>`**: Adds an integer ID to callbacks, useful for handling multiple sockets (e.g., a multi-port memory controller) with a single function.

## 5. Key Takeaways
1.  **Interoperability**: Allows a simple blocking target (easy to write) to work with a complex non-blocking initiator (high performance) without user code changes.
2.  **Conversion Cost**: The `nb` -> `b` conversion involves spawning a thread, which has a performance penalty. For maximum speed in AT models, native `nb_transport` should be used.
