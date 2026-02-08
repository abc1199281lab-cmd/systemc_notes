# SystemC Resolved Signal 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_signal_resolved.cpp`

## 1. 概述
`sc_signal_resolved` 是用於 `sc_logic` 的特化通道，支援 **多重寫入器 (multiple writers)** (匯流排建模)。它使用解析表解決衝突值 (例如，'0' 和 '1' 同時驅動)。

## 2. 解析機制

### 2.1 解析表 (`sc_logic_resolution_tbl`)
定義驅動兩個 `sc_logic` 值結果的 4x4 表格：
- `0` + `0` -> `0`
- `0` + `1` -> `X` (衝突)
- `0` + `Z` -> `0` (主導)
- `1` + `1` -> `1`
- ...

### 2.2 追蹤驅動器
- **`write(val)`**: 不像 `sc_signal` 儲存單一新值，`sc_signal_resolved` 維護一個數值清單 (`m_val_vec`)，每個寫入行程一個 (`m_proc_vec`)。
- 這允許它記住 *每個* 行程在 delta cycle 期間驅動什麼。

### 2.3 更新
- **`update()`**: 呼叫 `sc_logic_resolve` 使用解析表將驅動值向量縮減為單一最終值。

---

## 3. 關鍵重點
1.  **Wire-AND/OR Modeling**: 對於建模三態匯流排 (tri-state buses) 或多個元件可能同時驅動線路的雙向線路至關重要。
