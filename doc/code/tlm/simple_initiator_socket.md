# TLM Simple Initiator Socket Analysis

> **File**: `ref/systemc/src/tlm_utils/simple_initiator_socket.h`

## 1. Overview
The `simple_initiator_socket` is a utility class in `tlm_utils` that simplifies the implementation of initiators.

## 2. Problem It Solves
Standard TLM sockets require the owner module to inherit from `tlm_bw_transport_if` and implement `nb_transport_bw` and `invalidate_direct_mem_ptr`. This leads to multiple inheritance and name collisions if a module has multiple initiator sockets.

## 3. Mechanism
- **Callback Registration**: Instead of inheritance, `simple_initiator_socket` allows the user to register a member function as the callback for backward transport.
  ```cpp
  socket.register_nb_transport_bw(this, &MyModule::my_nb_transport_bw);
  ```
- **Internal Adapter**: The socket contains an internal `process` class that inherits from the BW interface and delegates calls to the registered callback.

## 4. Variants
- **`simple_initiator_socket<MODULE>`**: Standard version.
- **`simple_initiator_socket_tagged<MODULE>`**: For use when a module has multiple sockets bound to the *same* callback. The callback receives an integer `id` to identify which socket trigged the call.
  ```cpp
  void my_nb_transport_bw(int id, ...);
  ```

## 5. Key Takeaways
1.  **Cleaner Code**: Removes the need for modules to inherit from transport interfaces.
2.  **Multi-Socket Support**: The tagged version makes it easy to handle multiple ports with a single callback function.
