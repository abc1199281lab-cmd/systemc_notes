# SystemC Spawn Options 分析

> **檔案**: `ref/systemc/src/sysc/kernel/sc_spawn_options.cpp`

## 1. 概述
提供 `sc_spawn_options` 的實作，允許使用者設定透過 `sc_spawn` 建立的動態行程。

## 2. 機制
它主要充當行程建立時套用的設定容器。這裡實作最複雜的部分是 **重置訊號註冊**。

### 2.1 重置註冊 (`sc_spawn_reset`)
- 由於 `sc_spawn` 動態建立行程，我們無法在 elaboration 期間輕易使用靜態 `reset_signal_is`。
- `sc_spawn_options` 允許使用者指定重置訊號 (`async_reset_signal_is`, `reset_signal_is`)。
- 這些儲存在向量 `m_resets` 中。
- **`specify_resets()`**: 通常由核心在行程建立後但執行前呼叫，以實際將新行程註冊到 `sc_reset` 系統中。

---

## 3. 關鍵重點
1.  **動態組態 (Dynamic Configuration)**: 允許動態行程擁有與靜態行程幾乎完全相同的特性 (敏感度清單、dont_initialize、堆疊大小和重置)。
