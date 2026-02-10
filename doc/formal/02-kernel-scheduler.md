# 2026-02-08-kernel-scheduler

## 1. SystemC 排程演算法概論
SystemC 的排程器（Scheduler）是一個非搶佔式（Non-preemptive）的協同多工系統。它負責管理行程（Processes）的執行順序、處理事件觸發以及更新訊號狀態。

排程的核心邏輯封裝在 `sc_simcontext::crunch()` 函式中，其目標是推動模擬時間並處理所有在當前或未來時間點發生的活動。

## 2. 模擬的五個核心階段
排程器依照 IEEE 1666 標準定義的五個階段循環運作：

### A. 初始化階段 (Initialization Phase)
- 模擬開始時，除了明確呼叫 `dont_initialize()` 的行程外，所有的行程都會被加入「可執行隊列」（Runnable Queue）。
- 排程器會執行一次 `Update` 階段，確保初始的訊號值被傳遞。

### B. 評估階段 (Evaluation Phase)
- 排程器從可執行隊列中取出行程並執行。
- 行程執行時可能觸發「立即通知」（Immediate Notification），這會導致其他行程立即變為可執行並加入隊列。
- **注意**：同一 Delta Cycle 內的執行順序是不確定的（Unspecified Order）。

### C. 更新階段 (Update Phase)
- 當評估隊列清空後進入此階段。
- 呼叫所有已註冊的原始頻道（Primitive Channels）的 `update()` 方法（如 `sc_signal` 的新值寫入）。
- 這是解決競爭條件（Race Condition）的關鍵機制。

### D. Delta 通知階段 (Delta Notification Phase)
- 處理 `next_trigger()` 或 `sc_event::notify(SC_ZERO_TIME)` 觸發的事件。
- 被觸發的行程加入可執行隊列。如果隊列不為空，則返回 **評估階段**，形成一個新的 **Delta Cycle**。

### E. 時間通知階段 (Timed Notification Phase)
- 當所有的 Delta Cycle 都執行完畢後，如果沒有任何可執行行程，模擬時間會推進到下一個預定的事件時間。
- 處理 `sc_event::notify(time)` 觸發的事件，並返回 **評估階段**。

## 3. 核心原始碼追蹤 (`sc_simcontext.cpp`)
在 `crunch()` 函式中，可以看到以下邏輯：
```cpp
while ( true ) {
    // EVALUATE PHASE
    m_execution_phase = phase_evaluate;
    while( !m_runnable->is_empty() ) {
        // 取出並執行 method 或 thread
        ...
    }

    // UPDATE PHASE
    m_execution_phase = phase_update;
    m_prim_channel_registry->perform_update();

    // DELTA NOTIFICATION PHASE
    m_execution_phase = phase_notify;
    trigger_delta_events(); 
    
    if( m_runnable->is_empty() ) break; // 進入下一個時間點或結束
}
```

## 4. 關鍵術語
- **Runnable Queue**：存放當前準備好執行的行程。
- **Delta Cycle**：模擬時間不增加，但狀態發生演變的小循環。
- **Starvation**：當沒有任何事件與可執行行程時，模擬結束。

---
*Source: ref/systemc/src/sysc/kernel/sc_simcontext.cpp*
