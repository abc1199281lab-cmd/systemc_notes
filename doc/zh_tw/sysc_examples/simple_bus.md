# SystemC Simple Bus 範例分析

> **來源**: `ref/systemc/examples/sysc/simple_bus`

## 1. 概述
**Simple Bus** 是一個經典的 SystemC 範例，展示了如何使用 **階層式通道 (Hierarchical Channels)** 和自定義介面來建模通訊匯流排。不同於使用 generic payloads 和 sockets 的 TLM 2.0，此範例使用明確的基於連接埠 (port-based) 的通訊和自定義介面類別。

## 2. 架構

### 2.1 匯流排通道 (`simple_bus`)
匯流排實作為繼承自 `sc_module` 並實作了三個介面的自定義通道：
1.  **阻塞式 (`simple_bus_blocking_if`)**: 支援 `burst_read` 和 `burst_write`。使用此介面的 Master 會被掛起 (`wait`) 直到交易完成。
2.  **非阻塞式 (`simple_bus_non_blocking_if`)**: 支援 `read`, `write` 和 `get_status`。Master 發起請求並稍後檢查其狀態。
3.  **直接式 (`simple_bus_direct_if`)**: 支援除錯存取 (`direct_read`, `direct_write`)，不消耗模擬時間。

### 2.2 仲裁 (`simple_bus_arbiter`)
仲裁器選擇哪個請求獲得匯流排的使用權。
-   **優先權方案 (Priority Scheme)**:
    1.  **鎖定突發 (Locked Burst)**: 進行中且鎖定的突發傳輸不可被中斷。
    2.  **獲授權鎖定 (Lock Granted)**: 在上一個週期獲得鎖定的 Master 保有該鎖定。
    3.  **靜態優先權**: `priority` ID 越小優先權越高。

### 2.3 Slave (`simple_bus_slave_if`)
Slave 必須實作 `simple_bus_slave_if`。
-   **原子操作**: 匯流排在 Slave 上呼叫 `read()` 或 `write()`。
-   **狀態**: Slave 回傳 `SIMPLE_BUS_OK`, `SIMPLE_BUS_WAIT` (匯流排在下一個週期重試) 或 `SIMPLE_BUS_ERROR`。

## 3. 執行流程 (阻塞式突發)
1.  **Master**: 呼叫 `bus_port->burst_read(priority, ...)`.
2.  **Bus (介面)**: 建立一個 `simple_bus_request`，將其推送到 `m_requests`，並呼叫 `wait(request->transfer_done)`。
3.  **Bus (主要動作)**:
    -   在時脈下降緣執行。
    -   呼叫 `get_next_request()`，該函式會詢問 **仲裁器 (Arbiter)**。
    -   如果獲得授權，呼叫 `handle_request()`。
4.  **Bus (處理請求)**:
    -   將地址映射到 Slave。
    -   呼叫 `slave->read/write()`。
    -   如果狀態為 `OK`，前進突發計數器。如果突發結束，通知 `transfer_done`。
5.  **Master**: 從 `wait()` 中喚醒。

## 4. 與 TLM 2.0 的對照
| 特性 | Simple Bus (傳統) | TLM 2.0 |
| :--- | :--- | :--- |
| **通訊方式** | 自定義介面與通道 | Generic Payload 與 Sockets |
| **計時** | 週期精確 (時脈驅動) | LT (零時間/標註) 或 AT (階段式) |
| **互通性** | 低 (自定義介面) | 高 (標準協定) |
| **模擬速度** | 較慢 (每個週期都有等待) | 較快 (直接函式呼叫) |
