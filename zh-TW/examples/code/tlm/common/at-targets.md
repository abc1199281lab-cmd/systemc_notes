---
title: "AT Targets -- 非同步請求處理方"
linkTitle: "AT Targets"
weight: 5
description: "at_target_1_phase、at_target_1_phase_dmi、at_target_2_phase、at_target_4_phase 的實作解析"
---

## 概觀

AT (Approximately-Timed) target 使用 `nb_transport_fw` 接收請求，並透過 `nb_transport_bw` 回傳結果。不同的 target 實作了不同的 phase 協定，提供從簡單到完整的時序精確度選項。

### 軟體類比：HTTP 協定演進

| AT Target | 軟體類比 | 特點 |
|-----------|----------|------|
| `at_target_1_phase` | HTTP/1.0 | 收到請求立即回應，一步完成 |
| `at_target_2_phase` | HTTP/1.1 | 分 request 和 response 兩個階段 |
| `at_target_4_phase` | HTTP/2 | 完整的 request-accept-response-confirm 流程 |

## Phase 協定比較

```mermaid
sequenceDiagram
    participant I as Initiator
    participant T as Target

    rect rgb(230, 245, 230)
    Note over I,T: 1-Phase (TLM_COMPLETED)
    I->>T: nb_transport_fw(gp, BEGIN_REQ)
    T-->>I: TLM_COMPLETED + annotated delay
    end

    rect rgb(230, 235, 250)
    Note over I,T: 2-Phase (END_REQ + BEGIN_RESP)
    I->>T: nb_transport_fw(gp, BEGIN_REQ)
    T-->>I: TLM_UPDATED(phase=END_REQ)
    Note over T: PEQ 延遲後...
    T->>I: nb_transport_bw(gp, BEGIN_RESP)
    I->>T: nb_transport_fw(gp, END_RESP)
    T-->>I: TLM_COMPLETED
    end

    rect rgb(250, 235, 230)
    Note over I,T: 4-Phase (完整握手)
    I->>T: nb_transport_fw(gp, BEGIN_REQ)
    T-->>I: TLM_ACCEPTED
    T->>I: nb_transport_bw(gp, END_REQ)
    Note over T: 處理延遲後...
    T->>I: nb_transport_bw(gp, BEGIN_RESP)
    I->>T: nb_transport_fw(gp, END_RESP)
    T-->>I: TLM_COMPLETED
    end
```

## 共同架構

所有 AT target 都繼承 `tlm_fw_transport_if<>` 並實作以下介面：

- **`nb_transport_fw`** -- 處理來自 initiator 的前向請求
- **`begin_response_method`**（SC_METHOD）-- 從 PEQ 取出交易，透過 `nb_transport_bw` 發回結果
- **`m_response_PEQ`**（`peq_with_get`）-- Payload Event Queue，用於延遲排程回應
- **`m_target_memory`**（`memory` 類型）-- 實際的記憶體讀寫操作

## at_target_1_phase -- 混合模式 Target

**檔案**：`include/at_target_1_phase.h`, `src/at_target_1_phase.cpp`

這個 target 支援**兩種**回應模式，根據請求計數器切換：

- **前 19 筆**（`m_request_count % 20 != 0`）：直接回傳 `TLM_COMPLETED`（1-phase 模式）
- **每第 20 筆**：回傳 `TLM_UPDATED`（phase = END_REQ），然後排程 `BEGIN_RESP`（2-phase 模式）

### 工作流程

```mermaid
flowchart TD
    REQ["收到 BEGIN_REQ"] --> CHECK{"request_count % 20?"}
    CHECK -- "!= 0<br/>(大多數)" --> FAST["1-Phase 快速路徑"]
    FAST --> OP1["memory.operation(gp, delay)"]
    OP1 --> COMP["回傳 TLM_COMPLETED<br/>delay += accept_delay"]

    CHECK -- "== 0<br/>(每 20 筆)" --> SLOW["2-Phase 延遲路徑"]
    SLOW --> DELAY["memory.get_delay(gp, delay)"]
    DELAY --> PEQ["m_response_PEQ.notify(gp, delay)"]
    PEQ --> UPDATE["回傳 TLM_UPDATED<br/>phase = END_REQ"]
    PEQ --> BRM["begin_response_method 觸發"]
    BRM --> OP2["memory.operation(gp, delay)"]
    OP2 --> BW["nb_transport_bw(gp, BEGIN_RESP)"]
```

### 為什麼混合兩種模式？

這個設計展示了一個重要的 TLM 概念：**initiator 必須能處理來自同一個 target 的不同回應模式**。在真實硬體中，某些操作可能立即完成（如 cache hit），某些需要多步驟（如 cache miss 後的 memory 存取）。

## at_target_1_phase_dmi -- 1-Phase + DMI

**檔案**：`include/at_target_1_phase_dmi.h`, `src/at_target_1_phase_dmi.cpp`

與 `at_target_1_phase` 相同的邏輯，但額外支援 DMI。header 檔實際上重複使用了 `at_target_1_phase.h`（兩者的 header guard 相同：`__AT_TARGET_1_PHASE_H__`）。

`.cpp` 檔的實作與 `at_target_1_phase.cpp` 完全相同，是為了讓不同的範例連結不同的 `.o` 檔案。

## at_target_2_phase -- 標準兩階段 Target

**檔案**：`include/at_target_2_phase.h`, `src/at_target_2_phase.cpp`

所有請求都走 **2-phase 路徑**（不像 `at_target_1_phase` 有快速路徑）。

### 工作流程

```mermaid
sequenceDiagram
    participant I as Initiator
    participant T as at_target_2_phase

    I->>T: nb_transport_fw(gp, BEGIN_REQ, delay)
    Note over T: 計算延遲<br/>memory.get_delay(gp, delay)
    Note over T: 放入 PEQ<br/>m_response_PEQ.notify(gp, delay)
    T-->>I: TLM_UPDATED(phase=END_REQ, delay=accept_delay)

    Note over T: PEQ 延遲到期...
    Note over T: begin_response_method 觸發
    Note over T: memory.operation(gp, delay)
    T->>I: nb_transport_bw(gp, BEGIN_RESP, 0)

    alt TLM_COMPLETED
        I-->>T: TLM_COMPLETED(delay)
        Note over T: next_trigger(delay)
    else TLM_ACCEPTED
        I-->>T: TLM_ACCEPTED
        Note over T: next_trigger(m_end_resp_rcvd_event)
        I->>T: nb_transport_fw(gp, END_RESP)
        Note over T: m_end_resp_rcvd_event.notify()
        T-->>I: TLM_COMPLETED
    end
```

### 關鍵差異

與 `at_target_1_phase` 相比：

- **沒有**快速路徑（所有請求都走 PEQ）
- `nb_transport_fw` 在 `BEGIN_REQ` 時 **不執行** memory operation（只計算延遲）
- Memory operation 推遲到 `begin_response_method` 中執行

## at_target_4_phase -- 完整四階段 Target

**檔案**：`include/at_target_4_phase.h`, `src/at_target_4_phase.cpp`

實作完整的 4-phase 協定，是最精確的時序模型。

### 工作流程

```mermaid
sequenceDiagram
    participant I as Initiator
    participant T as at_target_4_phase

    I->>T: nb_transport_fw(gp, BEGIN_REQ, delay)
    Note over T: 排入 m_end_request_PEQ<br/>delay = delay + accept_delay
    T-->>I: TLM_ACCEPTED

    Note over T: PEQ 延遲到期...
    Note over T: end_request_method 觸發
    Note over T: 計算記憶體延遲
    Note over T: 排入 m_response_PEQ
    T->>I: nb_transport_bw(gp, END_REQ, 0)
    I-->>T: TLM_ACCEPTED

    Note over T: m_response_PEQ 延遲到期...
    Note over T: begin_response_method 觸發
    Note over T: memory.operation(gp, delay)
    T->>I: nb_transport_bw(gp, BEGIN_RESP, 0)

    alt TLM_COMPLETED
        I-->>T: TLM_COMPLETED(delay)
        Note over T: next_trigger(delay)
    else TLM_ACCEPTED
        I-->>T: TLM_ACCEPTED
        Note over T: 等待 END_RESP
        I->>T: nb_transport_fw(gp, END_RESP, delay)
        Note over T: m_end_resp_rcvd_event.notify(delay)
        T-->>I: TLM_COMPLETED
    end
```

### 架構特點

```mermaid
graph TD
    subgraph at_target_4_phase
        NB["nb_transport_fw<br/>(接收 BEGIN_REQ)"]
        ER_PEQ["m_end_request_PEQ"]
        ER["end_request_method<br/>(SC_METHOD)"]
        R_PEQ["m_response_PEQ"]
        BR["begin_response_method<br/>(SC_METHOD)"]
        MEM["m_target_memory"]
    end

    NB -- "1. BEGIN_REQ<br/>放入 PEQ" --> ER_PEQ
    ER_PEQ -- "2. 延遲到期" --> ER
    ER -- "3. 計算延遲<br/>放入 PEQ" --> R_PEQ
    ER -- "3. nb_transport_bw<br/>END_REQ" --> Initiator
    R_PEQ -- "4. 延遲到期" --> BR
    BR -- "5. 執行操作" --> MEM
    BR -- "5. nb_transport_bw<br/>BEGIN_RESP" --> Initiator
```

與 2-phase 的關鍵差異：

| 方面 | 2-phase | 4-phase |
|------|---------|---------|
| PEQ 數量 | 1 (`m_response_PEQ`) | 2 (`m_end_request_PEQ` + `m_response_PEQ`) |
| SC_METHOD 數量 | 1 (`begin_response_method`) | 2 (`end_request_method` + `begin_response_method`) |
| BEGIN_REQ 回傳 | `TLM_UPDATED`（phase = END_REQ） | `TLM_ACCEPTED`（END_REQ 稍後回傳） |
| END_REQ 發送方式 | 內嵌在 `nb_transport_fw` 回傳值 | 獨立的 `nb_transport_bw` 呼叫 |
| 時序分離 | request/response 兩個時間點 | accept/end_request/begin_response/end_response 四個時間點 |

## 四種 AT Target 的整體比較

| 特性 | 1-phase | 1-phase DMI | 2-phase | 4-phase |
|------|---------|-------------|---------|---------|
| Phase 數量 | 混合 1/2 | 混合 1/2 | 固定 2 | 固定 4 |
| PEQ 數量 | 1 | 1 | 1 | 2 |
| SC_METHOD 數量 | 1 | 1 | 1 | 2 |
| DMI 支援 | 否 | 是（stub） | 否 | 否 |
| 快速路徑 | 是（19/20 筆） | 是（19/20 筆） | 否 | 否 |
| 時序精確度 | 低~中 | 低~中 | 中 | 高 |
| Extension 支援 | 否 | 否 | 否 | 有（選用） |
| 適用場景 | 混合模式測試 | DMI + AT 測試 | 標準 AT 驗證 | 精確時序模型 |

### Extension 支援 (at_target_4_phase)

`at_target_4_phase` 在編譯時可選擇支援 `extension_initiator_id` 擴展：

```cpp
#ifdef USING_EXTENSION_OPTIONAL
extension_initiator_id *extension_pointer;
gp.get_extension(extension_pointer);
if (extension_pointer) {
    // 可讀取 extension_pointer->m_initiator_id
}
#endif
```

這展示了 TLM generic payload 的擴展機制 -- 類似 HTTP headers，允許在標準 payload 上附加自訂資訊。
