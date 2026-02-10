# 2026-02-08-kernel-module

## 1. sc_module 概論
`sc_module` 是 SystemC 結構化設計的基本建構塊。它是一個 C++ 類別，透過繼承 `sc_module`，我們可以定義硬體組件、系統模組或測試平台。

## 2. 宏與語法 (SC_MODULE & SC_CTOR)
為了簡化 C++ 的繁瑣語法，SystemC 提供了兩個核心宏：
- **`SC_MODULE(name)`**：定義一個繼承自 `sc_module` 的結構體。
- **`SC_CTOR(name)`**：定義模組的建構子，並自動傳遞 `sc_module_name`。

## 3. 生命周期回呼 (Life-cycle Callbacks)
`sc_module` 提供了多個虛擬函式，讓開發者可以在模擬的不同階段介入：

### A. Elaboration 階段
- **建構子**：實例化子模組、Port 與 Channel。
- **`before_end_of_elaboration()`**：在所有綁定完成前最後一次修改結構的機會。
- **`end_of_elaboration()`**：實例化與綁定正式完成。

### B. Simulation 階段
- **`start_of_simulation()`**：模擬循環（`sc_start`）開始前的最後準備工作。
- **`end_of_simulation()`**：模擬結束後的清理（如關閉追蹤檔案）。

## 4. 模組註冊與層級
當一個 `sc_module` 被實例化時：
1. 它會向 `sc_simcontext` 的 `sc_module_registry` 註冊自己。
2. 透過 `sc_module_name` 堆疊機制，自動建立「父-子」層級關係。
3. 每個模組都會獲得一個全域唯一的層級名稱（如 `top.cpu0.alu`）。

## 5. 原始碼參考
- `src/sysc/kernel/sc_module.h`: 類別定義與生命週期虛擬函式。
- `src/sysc/kernel/sc_module_name.h`: 處理層級名稱的堆疊機制。
- `src/sysc/kernel/sc_module_registry.h`: 管理所有模組實例。

---
*Source: ref/systemc/src/sysc/kernel/*
