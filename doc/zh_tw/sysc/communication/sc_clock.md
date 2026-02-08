# SystemC Clock 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_clock.cpp`

## 1. 概述
`sc_clock` 是一個預先定義的基本通道，產生週期性的時脈訊號。它繼承自 `sc_signal<bool>`。

## 2. 實作

### 2.1 初始化 (Initialization)
建構子計算：
- `m_posedge_time`: 高電位持續時間。
- `m_negedge_time`: 低電位持續時間。
- `m_start_time`: 何時開始。

### 2.2 行程產生 (Process Spawning)
- **`spawn_edge_method`**: 建立兩個內部 `SC_METHOD` 行程：一個用於正緣 (`posedge`)，一個用於負緣 (`negedge`)。
- 這些行程分別對 `m_next_posedge_event` 和 `m_next_negedge_event` 敏感。

### 2.3 執行迴圈 (Execution Loop)
1.  **Action**: 當 method 觸發時，它使用 `write()` 更新時脈的值。
2.  **Rescheduling**: 然後它通知 *下一個* 事件 (`notify_internal(m_period)`) 以在一個完整週期後再次觸發自己。
- 這建立了驅動時脈訊號的自我維持事件迴圈。

---

## 3. 關鍵重點
1.  **效率 (Efficiency)**: Clocks 實作為驅動訊號的標準 SystemC 行程，確保它們與排程器 (delta cycles 等) 完美整合。
