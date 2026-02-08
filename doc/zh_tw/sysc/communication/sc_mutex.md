# SystemC Mutex 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_mutex.cpp`

## 1. 概述
`sc_mutex` (互斥鎖) 是一個基本通道，提供對資源的互斥存取。它實作 `sc_mutex_if`。

## 2. 實作

### 2.1 狀態變數
- `m_owner`: 儲存目前持有鎖的 thread 的 `sc_process_b*`，`0` 表示它是自由的。
- `m_free`: 一個 `sc_event`，用於在 mutex 解鎖時喚醒等待的 threads。

### 2.2 機制
- **`lock()`**:
    - 如果 `m_owner` 符合目前的 process，它返回 0 (可重入/安全)。
    - 如果 `in_use()`，它呼叫 `wait(m_free)` 來阻塞目前的 thread。
    - 一旦自由，它將 `m_owner` 設定為目前的 process。
- **`unlock()`**:
    - 驗證有效擁有者。
    - 設定 `m_owner = 0`。
    - 呼叫 `m_free.notify()` 以喚醒 *所有*等待的 threads。(注意：由於 SystemC 的協作式非搶占排程，下一個 delta cycle/step 中只有一個會開始執行，但它們會競爭鎖)。

---

## 3. 關鍵重點
1.  **執行緒親和性 (Thread Affinity)**: Mutexes 由 threads (`sc_thread_process`) 擁有。Method processes 不能呼叫 `lock()` 因為它可能會等待。
2.  **RAII**: 標頭檔 `sc_mutex_if.h` 定義了 `sc_scoped_lock` 用於安全的例外處理。
