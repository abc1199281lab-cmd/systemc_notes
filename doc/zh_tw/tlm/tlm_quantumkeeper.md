# TLM Quantum Keeper 分析

> **檔案**: `ref/systemc/src/tlm_utils/tlm_quantumkeeper.h`

## 1. 概述
`tlm_quantumkeeper` 是 `tlm_utils` 中提供的一個工具類別，用來輔助發起者 (initiators) 實作時間解耦 (temporal decoupling)。

## 2. 狀態變數
- **`m_local_time`**: 發起者目前領先於 `sc_time_stamp()` 的時間偏移量。
- **`m_next_sync_point`**: 需要進行下一次同步的絕對 SystemC 時間 (`sc_time_stamp() + quantum`)。

## 3. 機制
- **`inc(t)`**: 累加區域時間延遲 (例如加入傳輸延遲、運算延遲)。
- **`need_sync()`**: 檢查累加的 `m_local_time` 加上當前時間是否已達到或超過 `m_next_sync_point`。
- **`sync()`**:
    1.  呼叫 `sc_core::wait(m_local_time)` 以實際推進 SystemC 時間。
    2.  將 `m_local_time` 重設為零。
    3.  計算新的 `m_next_sync_point`。

## 4. 使用模式
1.  發起者執行工作 (讀取/寫入)。
2.  發起者呼叫 `keeper.inc(delay)`。
3.  發起者檢查 `if (keeper.need_sync()) keeper.sync();`。

## 5. 關鍵重點
1.  **標準工具**: 雖然使用者可以實作自己的解耦邏輯，但 `tlm_quantumkeeper` 提供了一種標準化且穩健的方式。
2.  **命名空間 (namespace)**: 注意此類別位於 `tlm_utils` 中，而非 `tlm`。
