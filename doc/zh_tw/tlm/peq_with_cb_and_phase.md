# TLM PEQ with Callback and Phase 分析

> **檔案**: `ref/systemc/src/tlm_utils/peq_with_cb_and_phase.h`

## 1. 概述
`peq_with_cb_and_phase` 是 `tlm_utils` 中的一個工具類別，專為完全非阻塞模型 (AT 風格) 設計。不同於等待事件，它在交易 "成熟 (ripe)" 時觸發一個 **回調方法 (callback method)**。

## 2. 機制
- **回調 (Callback)**: 在建構子中接受一個函式指標 (方法)：`peq(this, &MyModule::my_callback)`。
- **內部佇列**:
    - `m_immediate_yield`: 用於立即通知。
    - `m_even_delta` / `m_uneven_delta`: 用於 delta-cycle 延遲 (處理特定的 SystemC delta cycle 邏輯)。
    - `m_ppq` (裝載優先權佇列, Payload Priority Queue): 用於計時通知。
- **`notify(trans, phase, delay)`**: 排定呼叫時間。
- **`fec()` (Fire Event Call)**: 一個對內部事件敏感的內部 `SC_METHOD`。觸發時，它會檢查佇列並為所有成熟的交易呼叫已註冊的回調。

## 3. 使用模式
用於 AT 起始者和目標，以在不阻塞執行緒的情況下模擬延遲。
```cpp
// 傳入路徑
peq.notify(trans, phase, delay);

// 回調
void my_callback(trans, phase) {
  // 處理階段轉換的邏輯，例如：發送 END_REQ 或 BEGIN_RESP
}
```

## 4. 關鍵重點
1.  **AT 建模**: 對於實作複雜狀態機的大約計時 (Approximately Timed, AT) 組件至關重要，因為這種組件中可能有多個交易同時進行中。
2.  **集中處理**: 將所有延時事件匯集到單一回調方法中，簡化了狀態管理。
