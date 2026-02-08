# TLM Loosely Timed (LT) Example Analysis

> **Source**: `ref/systemc/examples/tlm/lt` & `ref/systemc/examples/tlm/common`

## 1. Overview
This example demonstrates the **Loosely Timed (LT)** coding style in TLM 2.0. The LT style is used for high-performance software development (virtual prototyping) where timing accuracy is traded for simulation speed.

## 2. Key Characteristics
1.  **Blocking Transport**: The initiator uses `b_transport`.
2.  **Temporal Decoupling**: The initiator accumulates local time delay and only calls `wait()` when necessary (or after the transaction).
3.  **No Phases**: The transaction completes in a single function call.

## 3. Implementation Details

### 3.1 Initiator (`lt_initiator.cpp`)
The initiator runs in an `SC_THREAD`.
-   **Mechanism**:
    ```cpp
    // 1. Get generic payload
    transaction_ptr = request_in_port->read();

    // 2. Call blocking transport (does NOT consume simulation time yet)
    sc_time delay = SC_ZERO_TIME;
    initiator_socket->b_transport(*transaction_ptr, delay);

    // 3. Synchronize (Consume time)
    // The target added its processing time to 'delay'.
    // The initiator is responsible for calling wait().
    wait(delay); 
    ```
-   **Socket**: Uses `tlm_utils::simple_initiator_socket`.

### 3.2 Target (`lt_target.cpp`)
The target implements the transport logic.
-   **Mechanism**:
    ```cpp
    void lt_target::custom_b_transport(tlm_generic_payload &payload, sc_time &delay_time)
    {
        // 1. Perform operation (read/write memory)
        m_target_memory.operation(payload, mem_op_time);

        // 2. Annotate timing (Do NOT call wait)
        delay_time += mem_op_time + m_accept_delay;
    }
    ```
-   **Socket**: Uses `tlm_utils::simple_target_socket` and registers `custom_b_transport` as the callback.

## 4. Execution Flow
1.  **Start**: Initiator sends `b_transport` at T=0.
2.  **Target**: Executes command instantaneously, adds `10ns` (example) to delay. Returns.
3.  **Initiator**: Regains control. `delay` is now `10ns`. Calls `wait(10ns)`.
4.  **Simulation**: Time advances to T=10ns.

## 5. Key Takeaways
-   **Simplicity**: The code is linear and easy to read.
-   **Performance**: Context switching is minimized because the target does not `wait()`.
-   **Responsibility**: The **Initiator** is responsible for temporal synchronization (`wait`).
