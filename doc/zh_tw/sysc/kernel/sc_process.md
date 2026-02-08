# SystemC Process 分析: Base & Method

> **檔案**:
> - `ref/systemc/src/sysc/kernel/sc_process.cpp`
> - `ref/systemc/src/sysc/kernel/sc_method_process.cpp`

## 1. 概述
在 SystemC 中，任何 "執行" 的東西都是一個行程 (process)。
- **`sc_process_b` (Base)**: 所有行程類型的抽象基底類別。它處理共通的功能，如狀態管理、敏感度 (sensitivity) 和重置 (reset)。
- **`sc_method_process` (Method)**: 一種特定類型的行程，執行到完成且不能暫停。它對應到硬體中的組合邏輯 (combinational logic)。

---

## 2. `sc_process_b` (基底類別)
此類別管理模擬行程的生命週期和狀態。

### 2.1 行程狀態 (Process States)
一個行程可以處於多種狀態 (定義在 `sc_process.h` 但在此管理)：
- **`ps_normal`**: 正在執行或準備執行。
- **`ps_bit_suspended`**: 等待事件/時間。
- **`ps_bit_disabled`**: 被使用者停用 (即使觸發也不會執行)。
- **`ps_bit_zombie`**: 標記為刪除。

### 2.2 敏感度管理 (Sensitivity Management)
- **靜態敏感度 (Static Sensitivity)**: 在 elaboration 時間已知的事件 (例如 `sensitive << clk.pos()`)。
    - `add_static_event(e)`: 將行程註冊到事件。
- **動態敏感度 (Dynamic Sensitivity)**: 在模擬期間等待的事件 (例如 `wait(e)` 或 `next_trigger(e)`)。
    - `trigger_dynamic(e)`: 當動態事件觸發時被呼叫。

### 2.3 重置機制 (Reset Mechanism)
它處理同步和非同步重置 (`throw_reset`)。
- 當重置被斷言 (asserted) 時，它可以 "丟出" 一個例外來展開行程堆疊 (對於 threads) 或重新開始執行 (對於 methods)。

---

## 3. `sc_method_process` (SC_METHOD)
`SC_METHOD` 巨集會建立此類別的實例。

### 3.1 特性 (Characteristics)
- **無堆疊 (No Stack)**: 不像 threads，它沒有自己的堆疊。它在核心的堆疊上執行。
- **執行到完成 (Run-to-Completion)**: 它必須將控制權交還給核心。它不能呼叫 `wait()`。
- **`next_trigger()`**: Methods 使用 `next_trigger()` 而不是 `wait()` 來告訴核心 *何時* 再次執行它們。

### 3.2 執行流程 (`run_process`)
雖然在 `.cpp` 中不完全可見 (通常是 inline 或在 headers 中)，流程如下：
1.  **Trigger (觸發)**: 事件觸發 -> `trigger_dynamic()` 或 `trigger_static()`。
2.  **Runnable (可執行)**: 行程被推送到 `sc_simcontext` 的 runnable queue。
3.  **Execution (執行)**: `sc_simcontext::crunch` 取出它並呼叫其函式指標。
4.  **Completion (完成)**: 函式返回。

### 3.3 動態觸發 (`trigger_dynamic`)
此函式很複雜，因為它處理不同的觸發類型：
- `EVENT`: 單一事件。
- `AND_LIST`: 等待清單中 *所有* 事件。
- `OR_LIST`: 等待清單中 *任何* 事件。
- `TIMEOUT`: 等待時間。
- `EVENT_TIMEOUT`: 等待事件或時間。

當條件滿足時，它檢查行程是否被暫停/停用。如果有效，它將行程推送到 runnable queue：
```cpp
simcontext()->push_runnable_method(this);
```

---

## 4. 硬體 / RTL 對應

| SystemC (`sc_method_process`) | Verilog / SystemVerilog |
| :--- | :--- |
| `SC_METHOD` | `always @(sensitivity)` 或 `assign` |
| `next_trigger(e)` | 動態改變 `@(sensitivity)` 清單 (RTL 中少見，testbenches 中常見) |
| `dont_initialize()` | `always @(posedge clk)` (等待第一個邊緣), 相對於 `initial begin ... forever ... end` |

- **組合邏輯 (Combinational Logic)**: `SC_METHOD` 是模擬 `always @(*)` 或 `assign` 敘述的主要方式。
- **循序邏輯 (Sequential Logic - Synchronous)**: 如果對時脈邊緣敏感，也可以模擬，但 `SC_CTHREAD` 通常更受偏好用於類似合成的風格。

---

## 5. 關鍵重點
1.  **效率**: `sc_method_process` 比 `sc_thread_process` 輕量得多，因為它避免了上下文切換的開銷。盡可能使用它。
2.  **無狀態執行**: 由於它會返回，區域變數會遺失。狀態必須儲存在類別成員 (modules) 中。
3.  **例外處理**: 它具有強大的機制 (`throw_user`, `throw_reset`) 來處理錯誤和重置，即使它沒有持久的堆疊。
