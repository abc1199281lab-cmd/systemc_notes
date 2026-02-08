# SystemC Semaphore 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_semaphore.cpp`

## 1. 概述
`sc_semaphore` 是一個提供計數號誌功能的基本通道。它實作 `sc_semaphore_if`。

## 2. 實作

### 2.1 狀態變數
- `m_value`: 目前的號誌計數 (整數)。
- `m_free`: 當資源歸還 (`post`) 時通知的 `sc_event`。

### 2.2 機制
- **`wait()`**:
    - 檢查是否 `m_value > 0`。
    - 如果是 `0`，呼叫 `wait(m_free)` 來阻塞。
    - 在獲取後遞減 `m_value`。
- **`post()`**:
    - 遞增 `m_value`。
    - 呼叫 `m_free.notify()` 來喚醒等待的 threads。

---

## 3. 關鍵重點
1.  **阻塞 (Blocking)**: 像 mutex 一樣，`wait()` 允許限制資源存取排程 (例如，限制對匯流排的存取為 N 個 masters)。
