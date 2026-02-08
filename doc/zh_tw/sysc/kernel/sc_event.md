# SystemC Event 分析

> **檔案**: `ref/systemc/src/sysc/kernel/sc_event.cpp`

## 1. 概述
`sc_event` 是 SystemC 中基本的同步原語 (primitive)。它允許行程暫停執行並在特定條件發生時恢復。不像訊號 (signals)，事件沒有數值也沒有持續時間；它們是瞬間的。

## 2. 核心組件

### 2.1 `sc_event` 類別
- **目的**: 代表時間上的一個特定發生點。
- **狀態**: 包含等待它的行程清單：
    - `m_methods_static` / `m_methods_dynamic`
    - `m_threads_static` / `m_threads_dynamic`

### 2.2 通知機制 (`notify`)
`notify()` 方法是觸發事件的方式。它與 `sc_simcontext` 互動密切。

1.  **Immediate Notification (立即通知)** (`notify()` 無參數):
    - 導致 `trigger()` *立即* 發生 (如果在評估階段)。
    - **警告**: 如果不小心使用，可能會導致非確定性 (non-deterministic)。
2.  **Delta Notification (Delta 通知)** (`notify(SC_ZERO_TIME)`):
    - 將事件加入 `sc_simcontext` 的 delta event queue。
    - 它將在 *下一個* delta cycle (Update/Notify 階段) 開始時觸發。
3.  **Timed Notification (定時通知)** (`notify(time)`):
    - 加入 timed event queue。
    - 當模擬時間推進時觸發。

### 2.3 觸發過程 (`trigger`)
當事件觸發 (被 `sc_simcontext` 從佇列中取出) 時，`trigger()` 被呼叫：
1.  **Iterates (迭代)**: 遍歷所有靜態和動態等待行程清單。
2.  **Activates (活化)**:
    - 對於 `SC_METHOD`: 呼叫 `trigger_static` 或 `trigger_dynamic` (推送到 runnable queue)。
    - 對於 `SC_THREAD`: 在 `wait()` 之後恢復執行。
3.  **Cleanups (清理)**: 動態清單在觸發後被清除 (因為 `wait(e)` 是一次性的敏感度)。

### 2.4 事件清單 (`sc_event_list`)
處理如 `wait(e1 | e2)` 或 `wait(e1 & e2)` 的互動。
- **`OR_LIST`**: 第一個觸發的事件恢復行程，並將行程從其他事件的清單中移除。
- **`AND_LIST`**: 行程維護一個計數器。只有當 *所有* 事件都觸發時才恢復 (通常與 `SC_JOIN` 一起使用)。

---

## 3. 硬體 / RTL 對應

| SystemC (`sc_event`) | Verilog / SystemVerilog |
| :--- | :--- |
| `sc_event e;` | `event e;` |
| `e.notify()` | `-> e;` |
| `wait(e)` | `@(e);` |
| `wait(e1 | e2)` | `@(e1 or e2);` |
| `notify(SC_ZERO_TIME)` | 非阻塞賦值邏輯 (某種程度上)，或只是 `@(event)` 在下一個 NBA region 執行。 |

- **概念**: 事件是 `sc_signal` 的底層引擎。當訊號改變數值時，它有效地在內部事件 (`default_event()`) 上呼叫 `notify()`，這會喚醒敏感度清單。

---

## 4. 關鍵重點
1.  **無持續時間 (No Duration)**: 事件是一個時間點。如果你在 `e.notify()` 發生 *之後* 才 `wait(e)`，你會錯過它。
2.  **記憶體管理**: `sc_event` 的清理處理得很小心，特別是當行程死亡或事件是事件清單的一部分時。
3.  **核心事件 (Kernel Event)**: 有特殊的 "核心事件" (前綴)，用於內部同步，不會顯示在物件階層中。
