# TLM PEQ with Get 分析

> **檔案**: `ref/systemc/src/tlm_utils/peq_with_get.h`

## 1. 概述
`peq_with_get` (帶有 Get 的裝載事件佇列, Payload Event Queue with Get) 是 `tlm_utils` 命名空間中的一個工具類別。它提供了一種機制來延遲處理交易 (transactions)，直到特定時間點，從而允許行程 "等待 (wait)" 交易。

## 2. 機制
- **基於事件**: 它具有一個內部 `sc_event m_event`，當交易變得可用時 (由於時間推進) 會發送通知。
- **`notify(trans, t)`**: 將交易插入佇列中，排定於時間 `t` (相對於 `sc_time_stamp()`) 處理。它會呼叫 `m_event.notify(t)` 以在正確的時間喚醒消費者。
- **`get_next_transaction()`**:
    - 檢查是否有任何事件安排在當前模擬時間 (或更早)。
    - 如果有，則回傳該交易並將其從佇列中移除。
    - 如果沒有，則回傳 `NULL`。
    - 如果有未來的事件，它會針對到下一個事件的增量時間重新通知 `m_event`。

## 3. 使用模式
這通常用於需要接受非阻塞呼叫 (non-blocking calls) 但以阻塞/循序方式處理它們的模型中的 `SC_THREAD` 行程。
```cpp
// 在目標的 nb_transport_fw 中
peq.notify(trans, delay);
return TLM_ACCEPTED;

// 在目標的 SC_THREAD 中
void process_thread() {
  while(true) {
    wait(peq.get_event());
    while ((trans = peq.get_next_transaction()) != NULL) {
      // 處理交易
    }
  }
}
```

## 4. 關鍵重點
1.  **橋樑 (Bridge)**: 充當非阻塞領域 (傳入的 `nb_transport`) 與阻塞領域 (工作執行緒) 之間的橋樑。
