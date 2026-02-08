# SystemC Coroutine (Pthreads) 分析

> **檔案**: `ref/systemc/src/sysc/kernel/sc_cor_pthread.cpp`

## 1. 概述
此檔案使用 **POSIX Threads** (`pthreads`) 實作 SystemC 協程介面 (`sc_cor` 和 `sc_cor_pkg`)。這是 SystemC 行程切換的標準可攜式後端之一。

## 2. 核心機制

### 2.1 執行緒建立 (Thread Creation)
- **`create(...)`**: 產生一個新的真實作業系統執行緒 (`pthread_create`)。
- **Blocking**: 主執行緒立即在條件變數 (`m_create_cond`) 上阻塞，等待子執行緒啟動。
- **Child Handshake**: 子執行緒啟動，向主執行緒發送訊號 "我活著"，然後立即在其自己的條件變數 (`m_pt_condition`) 上阻塞，等待被 "切換進來"。
- **結果**: 即使你有一池的作業系統執行緒，但一次只有一個在執行 (協作式多工)。

### 2.2 上下文切換 (`yield`)
- **`yield(next_cor)`**: 從 `A` 切換到 `B`:
    1.  `A` 發送訊號給 `B` 的條件變數。
    2.  `A` 在 `A` 的條件變數上等待。
    3.  `B` 醒來並恢復執行。

---

## 3. 關鍵重點
1.  **嚴格順序性 (Strict Sequentiality)**: 即使它使用作業系統執行緒，SystemC 強制執行嚴格的序列化。一次只有一個執行緒持有 "活動" 令牌。
2.  **效能**: Pthreads 通常比使用者模式切換 (QuickThreads 或 Fibers) 更笨重，因為上下文切換涉及 OS 核心排程器。
