# SystemC Port 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_port.cpp`

## 1. 概述
`sc_port` 是模組與通道 (外部世界) 通訊的機制。它 "需要 (requires)" 一個介面。

## 2. 綁定機制 (Binding Mechanism)
綁定是將連接埠連接到介面 (通道) 或另一個連接埠的過程。

### 2.1 `sc_bind_info`
儲存綁定狀態。
- `vec`: 綁定元素的清單。
- `has_parent`: 如果此連接埠綁定到父連接埠 (階層式綁定)，則為 True。

### 2.2 `bind()`
- **bind(interface)**: 將介面加入綁定清單。
- **bind(parent_port)**: 將此連接埠連結到父連接埠。父連接埠最終必須綁定到一個介面。

### 2.3 `complete_binding()`
最關鍵的函式。它在 elaboration 結束時執行。
1.  **Flattening**: 它往上遍歷階層 (`first_parent()`) 以找到最終的來源介面。它將介面指標從父連接埠 "拉下 (pulls down)" 到葉連接埠。
2.  **Registration**: 呼叫 `interface->register_port(*this)` 以便通道知道此連線。
3.  **Sensitivity**: 透過在現在已綁定的介面上查找 event finder，解決任何靜態敏感度 (`sensitive << port`)。

## 3. 註冊表 (`sc_port_registry`)
追蹤模擬中的所有連接埠，以便在它們之上觸發生命週期回調 (`elaboration_done`, `start_of_simulation` 等)。

---

## 4. 關鍵重點
1.  **解析 (Resolution)**: 連接埠在模擬時間有效地 "消失"；它們變成直接指向通道介面的指標。`complete_binding` 階段確保了這種有效率的直接存取。
2.  **多重連接埠 (Multiports)**: 程式碼處理 `max_size` 和策略 (`SC_ONE_OR_MORE_BOUND` 等) 以支援連接到多個通道的連接埠。
