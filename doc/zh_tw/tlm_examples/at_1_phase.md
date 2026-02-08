# TLM 大約計時 (Approximately Timed, AT) 範例分析

> **來源**: `ref/systemc/examples/tlm/at_1_phase` & `ref/systemc/examples/tlm/common`

## 1. 概述
此範例展示了 **大約計時 (Approximately Timed, AT)** 編碼風格。AT 用於架構探索和性能建模，在這些場景中，通訊延遲和爭用是關注的重點。

## 2. 關鍵特性
1.  **非阻塞傳輸 (Non-Blocking Transport)**: 使用 `nb_transport_fw` 和 `nb_transport_bw`。
2.  **階段 (Phases)**: 明確地對 `BEGIN_REQ`, `END_REQ`, `BEGIN_RESP`, `END_RESP` 等階段進行建模。
3.  **反向路徑 (Backward Path)**: 目標會回傳呼叫發起者。
4.  **裝載事件佇列 (Payload Event Queues, PEQ)**: 用於將行程執行與回調計時解耦。

## 3. 實作細節

### 3.1 發起者 (`select_initiator.cpp`)
發起者發送請求並實作狀機來處理協定。
-   **正向路徑**:
    ```cpp
    // 發送 BEGIN_REQ
    tlm_phase phase = tlm::BEGIN_REQ;
    status = initiator_socket->nb_transport_fw(*trans, phase, delay);
    ```
-   **回傳處理**: 檢查狀態是否為 `TLM_ACCEPTED` (目標等待中)、`TLM_UPDATED` (目標狀態已變更) 或 `TLM_COMPLETED` (交易完成)。
-   **反向路徑 (`nb_transport_bw`)**:
    -   接收 `BEGIN_RESP` (回應有效，開始回應階段)。
    -   使用 PEQ (`m_send_end_rsp_PEQ`) 排定 `END_RESP`。

### 3.2 目標 (`at_target_1_phase.cpp`)
目標對延遲和管線 (pipelining) 進行建模。
-   **正向路徑 (`nb_transport_fw`)**:
    -   接收 `BEGIN_REQ`。
    -   計算延遲。
    -   將交易放入 `m_response_PEQ` (模擬內部處理延遲)。
    -   回傳 `TLM_UPDATED` 並帶有 `END_REQ` (確認請求)。
-   **回應處理 (`begin_response_method`)**:
    -   由 `m_response_PEQ` 觸發。
    -   呼叫 `nb_transport_bw` 並帶有 `BEGIN_RESP` 以將資料傳回發起者。

## 4. 執行流程 (4 階段)
1.  **發起者**: `nb_transport_fw(BEGIN_REQ)` -> 目標。
2.  **目標**: 內部排隊。回傳 `TLM_UPDATED(END_REQ)`。
3.  **目標 PEQ**: 延遲後觸發。呼叫 `nb_transport_bw(BEGIN_RESP)`。
4.  **發起者**: 內部排隊。回傳 `TLM_ACCEPTED`。
5.  **發起者 PEQ**: 觸發。呼叫 `nb_transport_fw(END_RESP)`。
6.  **目標**: 完成交易。

## 5. 關鍵重點
-   **複雜性**: 由於狀態機和反向路徑，複雜性顯著高於 LT。
-   **精確度**: 精確地模擬了管線階段和匯流排仲裁邏輯。
-   **PEQ**: 對於防止堆疊溢位 (回調的無限遞迴) 和建模精確的時序點至關重要。
