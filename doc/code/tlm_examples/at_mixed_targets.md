# TLM AT Mixed Targets Example Analysis

> **Source**: `ref/systemc/examples/tlm/at_mixed_targets` & `ref/systemc/examples/tlm/common`

## 1. Overview
This example demonstrates a system where an Approximately Timed (AT) initiator interacts with multiple AT targets, each implementing a different phase protocol (1-phase, 2-phase, and 4-phase). This highlights the interoperability of TLM 2.0 components.

## 2. Components

### 2.1 Initiator (`select_initiator.cpp`)
The initiator is capable of handling responses from all three target types.
-   **Mechanism**: It maintains a map (`m_waiting_bw_path_map`) to track the state of each outstanding transaction.
-   **State Tracking**:
    -   `Rcved_UPDATED_enum`: Expects 2-phase or early completion.
    -   `Rcved_ACCEPTED_enum`: Expects 4-phase (waiting for `BEGIN_RESP` or `END_REQ`).

### 2.2 Targets

#### A. 1-Phase Target (`at_target_1_phase.cpp`)
-   **Behavior**: Completes transaction immediately in `nb_transport_fw`.
-   **Return**: `TLM_COMPLETED`.
-   **Use Case**: Fast models or register accesses where no delay modeling is needed.

#### B. 2-Phase Target (`at_target_2_phase.cpp`)
-   **Behavior**:
    1.  Receives `BEGIN_REQ`. Returns `TLM_UPDATED` (Phase changes to `END_REQ`).
    2.  Wait for internal delay.
    3.  Calls `nb_transport_bw` with `BEGIN_RESP`.
-   **Protocol**: Request -> Accepted/Updated -> Response -> Completed. Valid for targets that can accept the request immediately but need time to process data.

#### C. 4-Phase Target (`at_target_4_phase.cpp`)
-   **Behavior**:
    1.  Receives `BEGIN_REQ`. Returns `TLM_ACCEPTED`.
    2.  Calculates delay. Calls `nb_transport_bw` with `END_REQ` (Request Phase Ends).
    3.  Processes data. Calls `nb_transport_bw` with `BEGIN_RESP` (Response Phase Begins).
    4.  Receiver (Initiator) sends `END_RESP` (Response Phase Ends).
-   **Protocol**: Full handshake. Explicitly models bus arbitration and pipeline stages for both Request and Response paths.

## 3. Interoperability Key
The key to mixing these targets is the **Initiator's robust state machine**. It does not assume a specific return value from `nb_transport_fw`. Instead, it invokes the correct handler based on whether the target returns `COMPLETED`, `UPDATED`, or `ACCEPTED`.

## 4. Key Takeaways
-   **Flexibility**: TLM 2.0 allows mixing components with different timing fidelities in the same simulation.
-   **Complexity Management**: The initiator bears the burden of supporting multiple protocols if it wants to be generic.
-   **Performance**: 1-phase is fastest, 4-phase is slowest (most callback overhead) but most accurate.
