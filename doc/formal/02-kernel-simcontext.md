# 2026-02-08-kernel-simcontext

## 1. sc_simcontext 的職責
`sc_simcontext` 是 SystemC 模擬核心的中樞神經系統。它封裝了模擬過程中的所有全域狀態，並管理著從實例化（Elaboration）到模擬結束（Post-simulation）的完整生命週期。

每個 SystemC 模擬通常至少有一個 `sc_simcontext` 實例。透過 `sc_get_curr_simcontext()` 可以取得當前的上下文。

### 核心職責包括：
- **Registry 管理**：維護 Module、Port、Export、Channel 以及 Process 的註冊表。
- **生命週期控制**：管理 `sc_start()`、`sc_stop()`、`initialize()` 等關鍵流程。
- **時間與 Delta 管理**：追蹤模擬時間（Time Stamp）與 Delta Cycle 的計數。
- **Process 管理**：負責 `SC_METHOD`、`SC_THREAD` 的建立、排程與執行。
- **層級與命名**：協助 `sc_object` 管理層級名稱與唯一名稱的生成。

## 2. 關鍵組件與註冊表
`sc_simcontext` 內部整合了多個註冊表類別，用於管理不同的 SystemC 對象：
- **`sc_object_manager`**：管理所有的 `sc_object`，負責處理層級關係。
- **`sc_module_registry`**：追蹤所有的 `sc_module` 實例。
- **`sc_port_registry` / `sc_export_registry`**：管理 Port 與 Export 的綁定（Binding）狀態。
- **`sc_prim_channel_registry`**：管理原始頻道（Primitive Channels），負責處理 `update()` 階段的更新請求。
- **`sc_process_table`**：儲存所有的行程（Process）。

## 3. 模擬狀態與階段 (sc_status & sc_stage)
`sc_simcontext` 維護著模擬的當前狀態，這對於許多 SystemC 宏（如 `SC_HAS_PROCESS`）的合法性檢查至關重要。

### 主要階段：
- **`SC_ELABORATION`**：構造函數執行期，進行模組實例化與連線。
- **`SC_BEFORE_END_OF_ELABORATION`**：實例化即將結束，可進行最後的動態調整。
- **`SC_END_OF_ELABORATION`**：實例化完成。
- **`SC_START_OF_SIMULATION`**：模擬即將開始。
- **`SC_RUNNING`**：模擬進行中（執行 `sc_start`）。
- **`SC_PAUSED`** / **`SC_STOPPED`**：模擬暫停或停止。
- **`SC_END_OF_SIMULATION`**：模擬結束後的清理階段。

## 4. 行程建立機制
當我們在 `sc_module` 中調用 `SC_METHOD` 或 `SC_THREAD` 時，底層會呼叫 `sc_simcontext` 的相關方法：
- `create_method_process()`
- `create_thread_process()`
- `create_cthread_process()`

這些方法負責分配記憶體、初始化堆疊（對於 Thread）並將其加入排程隊列。

## 5. 原始碼參考
- `src/sysc/kernel/sc_simcontext.h`: 類別定義與介面。
- `src/sysc/kernel/sc_simcontext.cpp`: 初始化、排程主循環與清理邏輯。
- `src/sysc/kernel/sc_status.h`: 狀態列舉定義。

---
*Source: ref/systemc/src/sysc/kernel/*
