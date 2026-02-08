# SystemC Signal Ports 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_signal_ports.cpp`

## 1. 概述
為連接到訊號的 ports 提供特化實作，特別是 `sc_in<bool>`, `sc_inout<bool>`, `sc_in<sc_logic>` 等。

## 2. 特化行為

### 2.1 `sc_in<bool>`
- **Tracing**: 包含邏輯 (`add_trace_internal`) 以允許將 port 加入 trace file。它有效地追蹤該 port 綁定的訊號。
- **Binding**: `vbind` 檢查父 port 是否為 `sc_in` 或 `sc_inout` 以允許階層式綁定。

### 2.2 `sc_inout<bool>`
- **Initialization**: `initialize(val)` 允許設定網路的初始值。如果已綁定，它寫入通道。如果尚未綁定，它緩衝數值 (`m_init_val`) 並在 elaboration 後寫入。

---

## 3. 關鍵重點
1.  **語法糖 (Syntactic Sugar)**: 這些特化存在是為了讓 `sc_in` 和 `sc_inout` 感覺像原生型別 (讀取/寫入)，同時保持 port 和 channel 的結構分離。
