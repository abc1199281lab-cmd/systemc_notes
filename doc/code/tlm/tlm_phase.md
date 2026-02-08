# TLM Phase Analysis

> **File**: `ref/systemc/src/tlm_core/tlm_2/tlm_generic_payload/tlm_phase.h`

## 1. Overview
`tlm_phase` represents the state of a transaction in the Base Protocol (Approximately Timed modeling).

## 2. Standard Phases
The `tlm_phase_enum` defines the four standard phases for the request-response cycle:
1.  **`BEGIN_REQ`**: Initiator sends a request.
2.  **`END_REQ`**: Target acknowledges receipt of the request.
3.  **`BEGIN_RESP`**: Target sends the response (and data for reads).
4.  **`END_RESP`**: Initiator acknowledges receipt of the response.

## 3. Extended Phases
Protocols often need more granular phases (e.g., "Address Phase", "Data Phase", "Snoop Phase").
- **`DECLARE_EXTENDED_PHASE(name)`**: A macro that creates a static singleton class for a new phase.
- **Mechanism**: The `tlm_phase` class can hold either a standard enum value OR a unique ID for an extended phase. This allows users to pass custom phases through the same `nb_transport` interface.

## 4. Key Takeaways
1.  **Finite State Machine**: These phases effectively define the states of the transaction FSM distributed between the initiator and target.
2.  **Extensibility**: Just like the Generic Payload, the Phase system is extensible to support custom protocols.
