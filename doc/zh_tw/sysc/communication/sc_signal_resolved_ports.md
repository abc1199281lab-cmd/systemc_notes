# SystemC Resolved Signal Ports 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_signal_resolved_ports.cpp`

## 1. 概述
定義專門連接到 `sc_signal_resolved` 通道的 ports (`sc_in_resolved`, `sc_inout_resolved`, `sc_out_resolved`)。

## 2. 型別安全 (Type Safety)
- **`end_of_elaboration()`**: 檢查綁定的介面是否確實是 `sc_signal_resolved*`。如果不是 (例如，連接到標準 `sc_signal<sc_logic>`)，它會報告錯誤 (`SC_ID_RESOLVED_PORT_NOT_BOUND_`)。
-這確保打算用於多重寫入器場景的 port 實際上連接到支援解析的通道。

---

## 3. 關鍵重點
1.  **明確意圖 (Explicit Intent)**: 使用這些 ports 宣告了 "此 port 可能是網路上眾多驅動器之一" 的意圖。
