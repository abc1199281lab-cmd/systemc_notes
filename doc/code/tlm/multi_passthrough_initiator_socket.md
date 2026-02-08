# TLM Multi-Passthrough Initiator Socket Analysis

> **File**: `ref/systemc/src/tlm_utils/multi_passthrough_initiator_socket.h`

## 1. Overview
The `multi_passthrough_initiator_socket` allows a single initiator socket to be bound to multiple targets. This is typically used in interconnects, routers, or buses.

## 2. Mechanism
- **Dynamic Binding**: Can bind to $N$ targets. The number of bindings is determined at elaboration time.
- **Indexing**: The socket maintains a vector of bound targets.
    - `socket[i]` returns the interface to the $i$-th target.
    - `socket.size()` returns the number of bound targets.

## 3. Callbacks
Since the backward path (callbacks from targets) needs to identify *which* target is responding, the registered callbacks include an index:
```cpp
// User callback signature
sync_enum_type nb_transport_bw(int id, transaction_type& trans, phase_type& phase, sc_time& t);

// Registration
socket.register_nb_transport_bw(this, &MyModule::my_nb_transport_bw);
```
The socket automatically assigns an ID (0 to N-1) to each connection and passes this ID to the callback.

## 4. Key Takeaways
1.  **Interconnects**: This is the building block for modeling buses and routers where one component talks to many.
2.  **No Conversion**: Unlike `simple_target_socket`, "passthrough" sockets do **NOT** perform blocking/non-blocking conversion. They are strictly for routing calls.
