# TLM Approximately Timed (AT) Example Analysis

> **Source**: `ref/systemc/examples/tlm/at_1_phase` & `ref/systemc/examples/tlm/common`

## 1. Overview
This example demonstrates the **Approximately Timed (AT)** coding style. AT is used for architectural exploration and performance modeling where communication delays and contention are interesting.

## 2. Key Characteristics
1.  **Non-Blocking Transport**: Uses `nb_transport_fw` and `nb_transport_bw`.
2.  **Phases**: Explicitly models phases like `BEGIN_REQ`, `END_REQ`, `BEGIN_RESP`, `END_RESP`.
3.  **Backward Path**: Targets call back into initiators.
4.  **Payload Event Queues (PEQ)**: Used to decouple process execution from callback timing.

## 3. Implementation Details

### 3.1 Initiator (`select_initiator.cpp`)
The initiator sends a request and implements a state machine to handle the protocol.
-   **Forward Path**:
    ```cpp
    // Send BEGIN_REQ
    tlm_phase phase = tlm::BEGIN_REQ;
    status = initiator_socket->nb_transport_fw(*trans, phase, delay);
    ```
-   **Return Handling**: Checks if status is `TLM_ACCEPTED` (target waiting), `TLM_UPDATED` (target state changed), or `TLM_COMPLETED` (transaction done).
-   **Backward Path (`nb_transport_bw`)**:
    -   Receives `BEGIN_RESP` (Response valid, starting response phase).
    -   Schedules `END_RESP` using a PEQ (`m_send_end_rsp_PEQ`).

### 3.2 Target (`at_target_1_phase.cpp`)
The target models latency and pipelining.
-   **Forward Path (`nb_transport_fw`)**:
    -   Receives `BEGIN_REQ`.
    -   Calculates delay.
    -   Puts transaction into `m_response_PEQ` (simulating internal processing latency).
    -   Returns `TLM_UPDATED` with `END_REQ` (Acknowledge request).
-   **Response Processing (`begin_response_method`)**:
    -   Triggered by `m_response_PEQ`.
    -   Calls `nb_transport_bw` with `BEGIN_RESP` to send data back to initiator.

## 4. Execution Flow (4-Phase)
1.  **Initiator**: `nb_transport_fw(BEGIN_REQ)` -> Target.
2.  **Target**: Queues internally. Returns `TLM_UPDATED(END_REQ)`.
3.  **Target PEQ**: Fires after delay. Calls `nb_transport_bw(BEGIN_RESP)`.
4.  **Initiator**: Queues internally. Returns `TLM_ACCEPTED`.
5.  **Initiator PEQ**: Fires. Calls `nb_transport_fw(END_RESP)`.
6.  **Target**: Completes transaction.

## 5. Key Takeaways
-   **Complexity**: Significantly more complex than LT due to state machines and backward paths.
-   **Accuracy**: Models pipeline phases and bus arbitration logic accurately.
-   **PEQs**: Essential for preventing stack overflows (infinite recursion of callbacks) and modeling accurate timing points.
