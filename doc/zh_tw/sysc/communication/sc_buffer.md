# SystemC Buffer 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_buffer.h`

## 1. 概述
`sc_buffer<T>` 是繼承自 `sc_signal<T>` 的基本通道。

## 2. 與 `sc_signal` 的差異
- **Signal**: *僅當* 新值與目前值不同時才觸發事件。
- **Buffer**: 在 *每次* 寫入時觸發事件，不管資料負載是否改變。

## 3. 實作
- **`write()`**: 總是設定新值並呼叫 `request_update()`。
- **`update()`**: 總是呼叫 `do_update()`，這會觸發數值改變事件。

---

## 4. 關鍵重點
1.  **使用案例**: 對於脈衝訊號傳遞或當寫入行為本身具有意義 (例如，刷新顯示、發送 "心跳") 時很有用，即使資料負載未改變。
