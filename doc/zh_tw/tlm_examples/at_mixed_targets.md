# TLM AT 混合目標 (Mixed Targets) 範例分析

> **來源**: `ref/systemc/examples/tlm/at_mixed_targets` & `ref/systemc/examples/tlm/common`

## 1. 概述
此範例展示了一個大約計時 (AT) 發起者與多個 AT 目標互動的系統，每個目標實作不同的階段協定 (1 階段、2 階段和 4 階段)。這突顯了 TLM 2.0 組件的互通性 (interoperability)。

## 2. 組件

### 2.1 發起者 (`select_initiator.cpp`)
發起者能夠處理來自所有三種目標類型的回應。
-   **機制**: 它維護一個映射表 (`m_waiting_bw_path_map`) 以追蹤每個待處理交易的狀態。
-   **狀態追蹤**:
    -   `Rcved_UPDATED_enum`: 預期為 2 階段或早期完成。
    -   `Rcved_ACCEPTED_enum`: 預期為 4 階段 (正在等待 `BEGIN_RESP` 或 `END_REQ`)。

### 2.2 目標

#### A. 1-階段目標 (`at_target_1_phase.cpp`)
-   **行為**: 在 `nb_transport_fw` 中立即完成交易。
-   **回傳**: `TLM_COMPLETED`。
-   **使用案例**: 快速模型或不需要延遲建模的暫存器存取。

#### B. 2-階段目標 (`at_target_2_phase.cpp`)
-   **行為**:
    1.  接收 `BEGIN_REQ`。回傳 `TLM_UPDATED` (階段變更為 `END_REQ`)。
    2.  等待內部延遲。
    3.  呼叫 `nb_transport_bw` 並帶有 `BEGIN_RESP`。
-   **協定**: 請求 -> 已接受/已更新 -> 回應 -> 已完成。適用於可以立即接受請求但需要時間處理資料的目標。

#### C. 4-階段目標 (`at_target_4_phase.cpp`)
-   **行為**:
    1.  接收 `BEGIN_REQ`。回傳 `TLM_ACCEPTED`。
    2.  計算延遲。呼叫 `nb_transport_bw` 並帶有 `END_REQ` (請求階段結束)。
    3.  處理資料。呼叫 `nb_transport_bw` 並帶有 `BEGIN_RESP` (回應階段開始)。
    4.  接收者 (發起者) 發送 `END_RESP` (回應階段結束)。
-   **協定**: 完整握手。明確地對請求和回應路徑的匯流排仲裁及管線階段進行建模。

## 3. 互通性關鍵
混合這些目標的關鍵在於 **發起者強大的狀態機**。它不假設 `nb_transport_fw` 的特定回傳值。相反，它根據目標回傳的是 `COMPLETED`、`UPDATED` 還是 `ACCEPTED` 來呼叫正確的處理程序。

## 4. 關鍵重點
-   **靈活性**: TLM 2.0 允許在同一個模擬中混合具有不同計時精確度的組件。
-   **複雜性管理**: 發起者承擔了支援多種協定以保持通用性的負擔。
-   **性能**: 1 階段最快，4 階段最慢 (回調開銷最大) 但最精確。
