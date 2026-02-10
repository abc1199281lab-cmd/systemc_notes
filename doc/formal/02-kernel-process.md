# 2026-02-08-kernel-process

## 1. SystemC 行程 (Process) 概論
在 SystemC 中，行程是執行行為的基本單元。不同於一般的 C++ 函式，行程是由 SystemC 核心（Kernel）進行註冊、管理與排程的。

所有的行程類別都繼承自基類 `sc_process_b`，並由 `sc_simcontext` 負責實例化。

## 2. 行程類型對照

### A. SC_METHOD (sc_method_process)
- **執行方式**：被排程器呼叫後從頭執行到尾，最後返回排程器。
- **堆疊 (Stack)**：與模擬核心共享同一個堆疊。
- **特性**：不可包含 `wait()`。效率高，適合描述組合邏輯或簡單的觸發行為。
- **敏感度**：僅支援靜態敏感度（Static Sensitivity）與透過 `next_trigger()` 實現的動態敏感度。

### B. SC_THREAD (sc_thread_process)
- **執行方式**：在獨立的執行上下文中運行。
- **堆疊 (Stack)**：擁有獨立的協程堆疊（Coroutine Stack）。
- **特性**：可以使用 `wait()` 語句掛起（Suspend）執行，並在事件發生時從掛起處繼續。適合描述複雜的循序邏輯（Sequential Logic）與測試平台。
- **底層機制**：使用 `sc_cor`（Coroutine）包裝來實現非搶佔式的上下文切換。

### C. SC_CTHREAD (sc_cthread_process)
- **執行方式**：一種特殊的 `SC_THREAD`，專門綁定於時脈邊緣（Clock Edge）。
- **特性**：主要用於綜合成 RTL，現在開發中較少見，通常建議使用 `SC_THREAD` 配合 `wait(clk.posedge_event())`。

## 3. 底層實作分析 (`sc_process_b`)
`sc_process_b` 維護了行程的核心狀態機：
- **`m_state`**：包括 `ps_bit_ready_to_run` (準備好執行)、`ps_bit_suspended` (已掛起)、`ps_bit_disabled` (已停用) 等。
- **`sc_entry_func`**：這是一個成員函式指標（Function Pointer），指向 `sc_module` 中定義的具體行為函式。

## 4. 協程 (Coroutine) 與上下文切換
對於 `SC_THREAD`，SystemC 並不使用作業系統級別的執行緒（OS Threads），而是使用使用者空間的協程（User-space Coroutines）：
- **`sc_cor`**：協程基類。
- **`sc_cor_pkg`**：協程包管理類。
- 當呼叫 `wait()` 時，當前行程會呼叫 `sc_cor_pkg::yield()`，將控制權交還給 `sc_simcontext` 的 `crunch()` 循環，而不銷毀當前的堆疊狀態。

## 5. 原始碼參考
- `src/sysc/kernel/sc_process.h`: `sc_process_b` 基礎類別定義。
- `src/sysc/kernel/sc_method_process.h`: Method 行程實作。
- `src/sysc/kernel/sc_thread_process.h`: Thread 行程實作。
- `src/sysc/kernel/sc_cor.h`: 協程底層抽象介面。

---
*Source: ref/systemc/src/sysc/kernel/*
