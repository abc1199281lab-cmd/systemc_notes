# SystemC Primitive Channel 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_prim_channel.cpp`

## 1. 概述
`sc_prim_channel` 是所有基本通道 (primitive channels) 的基底類別 (支援請求-更新評估階段的通道，如 `sc_signal`)。

## 2. 更新機制 (Update Mechanism)
基本通道需要將 "請求" (寫入) 與 "更新" (提交) 分開，以確保確定性 (delta cycles)。

### 2.1 註冊表 (`sc_prim_channel_registry`)
- 追蹤所有基本通道。
- **`perform_update()`**: 由核心在 delta cycle 結束時呼叫。它遍歷請求更新的通道清單並呼叫它們的 `update()` 方法。

### 2.2 非同步更新 (Async Updates)
- 支援來自外部執行緒的執行緒安全更新 (例如 `async_request_update`)。
- 使用 mutex 保護的清單 (`async_update_list`) 來安全地排隊更新，然後由核心在主執行緒中處理。

---

## 3. 關鍵重點
1.  **確定性 (determinism)**: `request_update()` -> `update()` 兩階段提交是 SystemC 對訊號模擬語義的核心。
2.  **執行緒安全 (Thread Safety)**: 此檔案包含特定邏輯 (`sc_scoped_lock`) 以安全地處理外部輸入。
