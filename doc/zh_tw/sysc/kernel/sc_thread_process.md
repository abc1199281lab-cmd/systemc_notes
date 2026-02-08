# SystemC Process 分析: Thread & CThread

> **檔案**:
> - `ref/systemc/src/sysc/kernel/sc_thread_process.cpp`
> - `ref/systemc/src/sysc/kernel/sc_cthread_process.cpp`

## 1. 概述
SystemC 中的 Threads 是 **擁有自己的堆疊 (stack)** 且 **可以暫停 (suspend)** 執行的行程 (在 wait 呼叫之間保留狀態)。
- **`sc_thread_process`**: 標準的 `SC_THREAD`。
- **`sc_cthread_process`**: 傳統/特化的 `SC_CTHREAD` (Clocked Thread)，主要用於高階合成 (HLS) 約束。

---

## 2. `sc_thread_process` (SC_THREAD)
這是行為建模的主力。

### 2.1 協程機制 (The Coroutine Mechanism)
不像 `SC_METHOD`，thread 使用協程 (coroutine)。
- `prepare_for_simulation()`: 呼叫 `simcontext()->cor_pkg()->create(...)` 為此 thread 分配一個獨立的堆疊。
- `sc_thread_cor_fn`: 靜態進入包裝函式 (entry wrapper function)。它捕捉例外並呼叫 `thread_h->semantics()` (最終呼叫使用者的函式)。

### 2.2 暫停與恢復 (Suspend & Resume)
- **`suspend_process()`**:
    - 將狀態標記為 `ps_bit_suspended`。
    - 有效率地從此 thread 進行上下文切換 (Context switches)。
- **`wait()` (隱式)**: 當使用者呼叫 `wait(e)` 時，核心在幕後呼叫 `suspend_process()` 並註冊敏感度。

### 2.3 例外處理與堆疊安全 (Exception Handling & Stack Safety)
由於 threads 有堆疊，跨越核心邊界丟出例外是很棘手的。
- **`throw_user`**: 允許一個行程將例外丟 *入* 另一個行程。
- **`throw_reset`**: 非同步重置是透過將例外注入 thread 以強制堆疊展開 (stack unwinding) (如果使用基於 `setjmp/longjmp` 的協程，則只是跳轉) 來實作的。
- **`SC_ALIGNED_STACK_`**: 程式碼包含特定架構的巨集 (x86, x64, Cygwin) 以確保堆疊針對現代編譯器/CPU 正確對齊 (16-byte)。

---

## 3. `sc_cthread_process` (SC_CTHREAD)
"Clocked Thread" 是 thread 的特化版本。

### 3.1 關鍵差異
- **繼承**: 它繼承自 `sc_thread_process`。
- **靜態敏感度**: 它 *僅* 對建構時定義的時脈邊緣敏感。
- **`dont_initialize`**: 它在建構子中呼叫 `m_dont_init = true`。`SC_CTHREAD` 通常不會在時間 0 (初始化階段) 執行，而是等待第一個時脈邊緣。

### 3.2 傳統地位
- 雖然在早期的 SystemC 合成中廣泛使用，但現代 HLS 工具通常接受帶有特定編碼風格 (無限迴圈 + 起始處 wait) 的 `SC_THREAD`。
- 然而，`SC_CTHREAD` 強制 "僅在時脈上等待" 有助於早期的合成工具。

---

## 4. 硬體 / RTL 對應

| SystemC (`sc_thread_process`) | Verilog / SystemVerilog |
| :--- | :--- |
| `SC_THREAD` | `initial begin ... end` 或帶有延遲的 `always` 區塊 |
| `wait(10, SC_NS)` | `#10;` |
| `wait(event)` | `@(event);` |
| `sc_cthread_process` | `always @(posedge clk)` |
| `wait()` (in CTHREAD) | 等待下一個有效的時脈邊緣 |

- **行為建模**: Threads 非常適合撰寫 testbenches (激勵產生)，如果你想要線性、循序的程式碼流程。
- **HLS**: 用於描述跨越多個時脈週期的複雜循序演算法。

---

## 5. 關鍵重點
1.  **上下文切換成本**: `SC_THREAD` 比 `SC_METHOD` 笨重，因為交換堆疊需要 CPU 時間。純組合邏輯請使用 Method。
2.  **堆疊大小**: 每個 thread 消耗記憶體。`SC_DEFAULT_STACK_SIZE` (通常是 64KB 或 128KB) 可以被覆寫，如果你有數千個 threads 或深度遞迴。
3.  **模擬速度**: 太多 threads 會因為堆疊交換造成的 cache pollution 而以降低模擬速度。
