# SystemC Stage Callback Registry 分析

> **檔案**: `ref/systemc/src/sysc/kernel/sc_stage_callback_registry.cpp`

## 1. 概述
實作 **模擬階段回調 (simulation stage callbacks)** 的註冊表。這個現代 SystemC 功能允許使用者和工具 (除錯器、追蹤器、檢查器) 註冊在模擬迴圈中特定點被呼叫的函式。

## 2. 階段 (`sc_stage`)
定義於 `sc_stage_callback_if.h`，這些位元遮罩 representing 執行階段：
- `SC_POST_END_OF_ELABORATION`
- `SC_POST_UPDATE` (在 request_update/update 階段之後)
- `SC_PRE_TIMESTEP` (在時間推進之前)
- `SC_POST_END_OF_SIMULATION`
- ... 以及其他。

## 3. 註冊機制
- **註冊**: `register_callback(cb, mask)` 為特定階段加入回調。
- **最佳化**: 為高頻率階段 (如 UPDATE 和 TIMESTEP) 維護單獨的向量 (`m_cb_update_vec`, `m_cb_timestep_vec`)，以避免在模擬迴圈的關鍵路徑期間遍歷整個清單。
- **驗證**: 檢查請求的遮罩是否有效，以及階段是否已經過去 (例如，試圖在模擬開始後註冊 Elaboration Done)。

---

## 4. 關鍵重點
1.  **儀表化 (Instrumentation)**: 這是建構非侵入式模擬控制和檢查工具的主要掛鉤 (hook)。
