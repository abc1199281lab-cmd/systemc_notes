# TLM Multi-Passthrough Target Socket Analysis

> **File**: `ref/systemc/src/tlm_utils/multi_passthrough_target_socket.h`

## 1. Overview
The `multi_passthrough_target_socket` allows multiple initiators to bind to a single target socket. This is commonly used in shared resources (like memory) or interconnects.

## 2. Mechanism
- **Dynamic Binding**: Accepts connections from $N$ initiators.
- **Indexing**: `socket[i]` provides access to the backward path of the $i$-th initiator.

## 3. Callbacks
Forward path calls (from initiators) need to identify the source. The registered callbacks include an index:
```cpp
// User callback signature
void b_transport(int id, transaction_type& trans, sc_time& t);

// Registration
socket.register_b_transport(this, &MyModule::my_b_transport);
```
When Initiator $K$ calls `b_transport`, the socket invokes `my_b_transport` with `id = K`.

## 4. Key Takeaways
1.  **Arbitration/Routing**: Essential for components that need to distinguish between multiple sources of traffic (checking IDs to arbitrate or map addresses).
2.  **Triviality**: Like its initiator counterpart, it does not convert transaction types. It simply adds the ID and forwards the call.
