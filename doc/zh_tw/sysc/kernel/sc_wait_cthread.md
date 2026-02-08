# SystemC Wait CThread 分析

> **檔案**: `ref/systemc/src/sysc/kernel/sc_wait_cthread.cpp`

## 1. 概述
此檔案實作專門用於 **`SC_CTHREAD`** (clocked threads) 的 `wait()` 重載。`SC_CTHREAD` 是一種傳統 (但仍支援) 的建構，其中 thread 預設對時脈邊緣敏感。

## 2. 關鍵函式

### 2.1 `wait(int n)`
- 等待 `n` 個時脈週期。
- **機制**: 呼叫 `sc_cthread_handle` 上的 `wait_cycles(n)`。
- **限制**: 只能從 `SC_CTHREAD` 呼叫。從 `SC_METHOD` 呼叫是錯誤的。

### 2.2 `at_posedge` / `at_negedge`
- 直到訊號發生特定邊緣時才繼續的工具函式。
- 實作為 `wait()` 呼叫的迴圈，直到訊號值符合預期的準位。
- **效率低落**: 這些忙碌等待迴圈 (`do { wait(); } while (...)`) 有效地跳過時脈週期，直到條件滿足。

### 2.3 `halt()`
- 無限期停止 `SC_CTHREAD` 的執行。
- 內部呼叫 `wait_halt()`。

---

## 3. 關鍵重點
1.  **可合成性 (Synthesizability)**: 這些建構 (`wait(n)`, `at_posedge`) 最初是為高階合成 (HLS) 流程設計的，以便輕易表達週期精確 (cycle-accurate) 行為。
