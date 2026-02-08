# SystemC Event Queue 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_event_queue.cpp`

## 1. 概述
`sc_event_queue` 允許同一個通道有多個待處理的通知，不像 `sc_event` 如果已經有針對同一時間的待處理通知則會忽略後續通知。

## 2. 機制

### 2.1 優先權佇列 (`m_ppq`)
- 儲存所有請求的通知時間。
- 依時間排序 (最早的優先)。

### 2.2 `notify(time)`
- 將時間推送到佇列上。
- 如果這是最早的事件，它通知底層的 `sc_event m_e`。

### 2.3 `fire_event`
- 一個對 `m_e` 敏感的 `SC_METHOD`。
- 從佇列中彈出最頂端的事件。
- 如果佇列中還有更多事件，它會針對當前時間與下一個事件時間之間的 delta 重新通知 `m_e`。

---

## 3. 關鍵重點
1.  **使用案例 (Use Case)**: 對於建模如封包到達佇列 (packet arrival queues) 的事物至關重要，其中多個封包可能在完全相同的模擬時間或非常接近的時間到達，而你不希望遺漏任何通知。
