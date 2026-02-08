# SystemC Wait 分析

> **檔案**:
> - `ref/systemc/src/sysc/kernel/sc_wait.cpp`
> - `ref/systemc/src/sysc/kernel/sc_wait_cthread.cpp`

## 1. 概述
`wait()` 是 SystemC 中 **行程同步 (Process Synchronization)** 的主要機制。它暫時暫停 thread 的執行，並將控制權交還給模擬核心 (`sc_simcontext::crunch`)。

## 2. `wait()` 外觀模式 (Facade)
`sc_wait.cpp` 中的全域 `wait()` 函式充當 **Facade** / **Dispatcher**。它們本身不包含暫停執行的邏輯；相反，它們委派給目前的行程 handle。

### 2.1 分派邏輯 (Dispatch Logic)
```cpp
void wait( const sc_event& e, sc_simcontext* simc ) {
    sc_curr_proc_handle cpi = simc->get_curr_proc_info();
    switch( cpi->kind ) {
    case SC_THREAD_PROC_:
        // 委派給 thread 實作
        reinterpret_cast<sc_thread_handle>( cpi->process_handle )->wait( e );
        break;
    case SC_CTHREAD_PROC_:
         // 委派給 cthread 實作 (加上週期處理)
         // ...
        break;
    default:
        // 錯誤: methods 不能 wait!
        SC_REPORT_ERROR( SC_ID_WAIT_NOT_ALLOWED_, ... );
    }
}
```

### 2.2 `next_trigger()`
對於 `SC_METHOD`，對應的函式是 `next_trigger()`。因為 methods 不能暫停 (沒有堆疊)，`next_trigger()` 只是告訴核心："當這個 method 結束時，讓它對 *這個* 事件敏感，以便下次執行。"

---

## 3. `SC_CTHREAD` 特性
`sc_wait_cthread.cpp` 為 Clocked Threads 增加了特化的 wait 函式：
- **`wait(N)`**: 等待 N 個時脈週期。這會呼叫 `wait_cycles(n)`。
- **`halt()`**: 無限期停止 thread (本質上是沒有參數的 `wait()`，但意圖表示 "完成")。
- **`at_posedge(sig)`**: 迴圈 `do { wait(); } while (...)` 直到訊號符合邊緣。這實際上是一個與時脈同步的 "輪詢迴圈 (poll loop)"。

---

## 4. 硬體 / RTL 對應

| SystemC | Verilog / SystemVerilog |
| :--- | :--- |
| `wait(10, SC_NS)` | `#10` |
| `wait(event)` | `@(event)` |
| `wait(N)` (CTHREAD) | `repeat(N) @(posedge clk)` |
| `next_trigger(e)` | 改變敏感度清單 (動態) |

### 4.1 合成影響
- **`wait()`** 在 `SC_THREAD` 中: 在高階合成 (HLS) 中通常暗示時脈邊界，有效地插入暫存器。
- **`wait(N)`**: 插入 N 個狀態的狀態機鏈 (等待 N 個時脈)。

---

## 5. 關鍵重點
1.  **上下文感知 (Context Aware)**: `wait()` 知道哪個行程正在呼叫它。如果你從 `SC_METHOD` 呼叫它，它會丟出執行期錯誤。
2.  **讓出 (Yielding)**: 每個 `wait()` 都會讓出執行。如果一個 thread 迴圈 `while(1) {}` 而沒有 `wait()`，它會掛起模擬 (餓死核心)。
3.  **靜態與動態 (Static vs Dynamic)**:
    - `sensitive << e` (Static): 在建構時定義，當在 `wait()` 時總是有效的。
    - `wait(e)` (Dynamic): 暫時覆蓋靜態敏感度 (取決於確切的 wait 類型，通常只是等待 `e`)。
