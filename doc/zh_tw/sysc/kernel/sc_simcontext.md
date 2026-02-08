# SystemC SimContext 分析

> **檔案**: `ref/systemc/src/sysc/kernel/sc_simcontext.cpp`

## 1. 概述
`sc_simcontext` 是 SystemC 模擬核心的 **中樞神經系統**。它是一個類似單例 (Singleton-like) 的物件 (雖然技術上可以存在多個實例)，負責管理：
1.  **全域模擬狀態 (Global Simulation State)**: 時間、Delta cycles、模擬狀態 (elaboration, running, paused)。
2.  **物件註冊表 (Object Registry)**: 追蹤所有的模組 (modules)、連接埠 (ports)、通道 (channels) 和行程 (processes)。
3.  **排程器 (Scheduler)**: `crunch()` 方法是離散事件模擬演算法的核心。
4.  **行程管理 (Process Management)**: 持有 `m_process_table` 以管理 method 和 thread。

---

## 2. 核心組件

### 2.1 排程迴圈: `crunch()`
這是整個核心中最關鍵的函式。它實作了 **Evaluate-Update-Notify** (評估-更新-通知) 的 delta cycle 迴圈。

```cpp
// crunch() 的簡化邏輯
void sc_simcontext::crunch( bool once ) {
    while (true) {
        // 1. EVALUATE PHASE (評估階段)
        // 執行所有可執行的行程 (methods 和 threads)
        // 這是使用者程式碼執行的地方！
        
        while (runnable_processes_exist) {
            execute_method_processes();
            execute_thread_processes();
        }

        // 2. UPDATE PHASE (更新階段)
        // 基本通道 (如 sc_signal) 在此更新其數值
        // 以確保改進的確定性 (避免 race conditions)。
        m_prim_channel_registry->perform_update();

        // 3. NOTIFICATION PHASE (通知階段)
        // 觸發在 evaluate/update 期間收到通知的事件
        // 這可能會讓新的行程變成可執行 (runnable) 狀態，以便在下一個 delta cycle 執行。
        trigger_delta_events();

        // 4. Time Advancement Check (時間推進檢查)
        // 如果沒有更多可執行的行程，也沒有更多的 delta events，
        // delta cycle 迴圈結束，時間透過 `sc_start` 推進。
    }
}
```

### 2.2 行程管理 (Process Management)
它使用 `sc_process_table` 來追蹤：
- **Method Processes** (`SC_METHOD`): 執行到完成，不能暫停 (suspend)。
- **Thread Processes** (`SC_THREAD`): 可以暫停 (`wait()`)，需要協程 (coroutine) 上下文切換。

### 2.3 物件註冊表 (Object Registries)
`sc_simcontext` 擁有多個註冊表來管理階層結構：
- `m_object_manager`: 一般物件管理。
- `m_module_registry`: 追蹤所有 `sc_module` 實例。
- `m_port_registry` / `m_export_registry`: 追蹤連接性。

---

## 3. 硬體 / RTL 對應

| SystemC (`sc_simcontext`) | 硬體 / Verilog 模擬器 |
| :--- | :--- |
| `sc_simcontext` | **Simulator Kernel** 本身 (引擎)。 |
| `crunch()` Loop | **Event-Driven Simulation Algorithm** (事件驅動模擬演算法)。 |
| `m_delta_count` | **Delta Cycle** 計數器 (虛擬時間步)。 |
| `m_curr_time` | **$time** (模擬時間)。 |
| `phase_evaluate` | **Active Region** (執行 `always` 區塊)。 |
| `phase_update` | **NBA Region** (非阻塞賦值更新 `<=`)。 |

### 3.1 "Delta Cycle" 概念
在 Verilog 中：
```verilog
always @(posedge clk) begin
    a <= b; // NBA
    c <= a; // 讀取 a 的舊值
end
```
在 SystemC 中，`sc_simcontext` 透過 **Evaluate-Update** 階段強制執行此行為。
1.  **Evaluate**: `a.write(b)` 被呼叫。新值被 *暫存 (staged)* 但尚未套用。
2.  **Update**: `sc_simcontext` 呼叫 `perform_update()`，將新值套用到 `a`。
這確保了 `c` 會讀取到 `a` 的舊值，如果它們在同一個 evaluation phase 中執行。

---

## 4. 關鍵重點
1.  **確定性 (Determinism)**: 排程器對 Evaluate 和 Update 階段的分離，使得 SystemC 具有週期精確度 (cycle-accurate) 和確定性，符合硬體行為。
2.  **集中化 (Centralization)**: SystemC 中的幾乎所有操作 (建立模組、建立訊號、通知事件) 最終都會回調到 `sc_simcontext` 進行註冊。
3.  **協程 (Coroutines)**: 它管理協程套件 (`m_cor_pkg`) 以支援 `SC_THREAD` 上下文切換，這是 "並發硬體行程" 的軟體實作。
