# SystemC Event Finder 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_event_finder.cpp`

## 1. 概述
`sc_event_finder` 是一個輔助類別，由 ports 內部使用以支援 `sensitive << port.pos();`。

## 2. 問題
當你寫 `sensitive << port.pos()` 時，port 可能尚未被綁定。因此，`port.pos()` 無法返回來自通道的實際事件，因為通道還不知道。

## 3. 解決方案
- `port.pos()` 返回 `sc_event_finder` 的特定實作。
- `sc_event_finder` 暫時持有對 port 的參考。
- 在 `complete_binding` 或敏感度被定案時，核心呼叫 finder 上的 `find_event()`。
- Finder 詢問 port 其綁定的介面，*然後* 從該介面檢索實際事件。

---

## 4. 關鍵重點
1.  **延遲解析 (Deferred Resolution)**: 這是處理建構時間 (敏感度宣告) 和 elaboration 時間 (綁定) 之間分離的經典 "延遲評估 (lazy evaluation)" 模式。
