# SystemC Signal 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_signal.cpp`

## 1. 概述
`sc_signal<T>` 是點對點通訊的基本通道。

## 2. 實作細節

### 2.1 事件 (Events) (`sc_signal_channel`)
- 使用 **延遲事件分配 (Lazy Event Allocation)**: Events (`m_change_event_p`, `m_posedge_event_p`) 僅在被請求時 (例如，行程等待它們時) 才分配。這為僅讀取但從未等待的訊號節省記憶體。

### 2.2 更新邏輯 (`do_update`)
- 由 `sc_prim_channel::perform_update` 在 delta 結束時呼叫。
- 檢查 `m_new_val != m_cur_val`。
- 如果改變：
    1.  更新 `m_cur_val`。
    2.  通知 `value_changed_event`。
    3.  通知 `posedge_event` 或 `negedge_event` (對於 bool/logic 訊號)。

### 2.3 寫入策略 (Writer Policies)
- 如果 `SC_ONE_WRITER` 策略啟用，則檢查多重驅動器。
- `sc_signal_invalid_writer` 報告衝突。

---

## 3. 關鍵重點
1.  **最佳化**: 延遲事件機制是一個重要的記憶體最佳化。
2.  **合規性**: 除非另有設定，否則嚴格執行單一驅動器規則。
