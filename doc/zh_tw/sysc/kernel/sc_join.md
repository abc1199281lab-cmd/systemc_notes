# SystemC Join 分析

> **檔案**: `ref/systemc/src/sysc/kernel/sc_join.cpp`

## 1. 概述
`sc_join` 實作 SystemC 中的 **Fork-Join** 模式。它允許一個行程產生動態 threads (使用 `sc_spawn`)，然後等待它們完成執行。

## 2. 機制
它使用 **觀察者模式 (Observer Pattern)** (透過 `sc_process_monitor`) 來追蹤 thread 生命週期。

### 2.1 註冊 (`add_process`)
當你加入一個行程到 join 物件時：
1.  它增加內部計數器 `m_threads_n`。
2.  它將自己註冊為該行程的 **觀察器 (monitor)** (`handle->add_monitor(this)`)。

### 2.2 監控 (`signal`)
`sc_join` 類別實作來自 `sc_process_monitor` 的 `signal()` 回調：
```cpp
void sc_join::signal(sc_thread_handle thread_p, int type) {
    if (type == sc_process_monitor::spm_exit) {
        // Thread 已死亡
        if ( --m_threads_n == 0 ) {
            // 所有被監視的 threads 都已死亡
            m_join_event.notify();
        }
    }
}
```

### 2.3 等待 (Waiting)
使用者通常呼叫 `wait(my_join.event())`。`m_join_event` 僅在最後一個被監視的 thread 退出時觸發。

---

## 3. 使用模式 (`SC_FORK` / `SC_JOIN`)
雖然 `sc_join` 可以手動使用，但它是 `SC_FORK ... SC_JOIN` 巨集 (或 `sc_spawn` + handle 向量) 的後端。

```cpp
// 概念用法
sc_join my_join;
sc_process_handle h1 = sc_spawn(...);
sc_process_handle h2 = sc_spawn(...);
my_join.add_process(h1);
my_join.add_process(h2);
wait(my_join.event()); // 阻塞直到 h1 和 h2 都終止
```

---

## 4. 硬體 / RTL 對應
這種模式在可合成的 RTL (通常是固定階層) 中較少見，但對應到：
- **Testbench Fork/Join**: SystemVerilog `fork ... join` 或 `fork ... join_any`。
- **HLS**: 必須在繼續之前重新同步的並行程式碼區段。

---

## 5. 關鍵重點
1.  **動態 vs 靜態**: `sc_join` 處理動態行程 (執行期誕生)。
2.  **零開銷監控**: 它使用 `sc_thread_process` 中內建的 monitor 指標，所以它不輪詢 (poll)；它是事件驅動的。
