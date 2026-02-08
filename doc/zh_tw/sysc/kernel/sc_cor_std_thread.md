# SystemC Coroutine (std::thread) 分析

> **檔案**: `ref/systemc/src/sysc/kernel/sc_cor_std_thread.cpp`

## 1. 概述
使用 C++11 `std::thread` 的 SystemC 協程現代實作。這類似於 Pthreads 實作，但使用標準 C++ API，使其更具可攜性且無須外部相依性。

## 2. 核心機制

### 2.1 握手邏輯 (Handshake Logic)
就像 Pthreads 版本一樣，它使用 mutex/condition variable 對 (`m_create_mutex`, `m_create_cond`) 來執行握手。
1.  **Main**: 建立 `std::thread`，等待。
2.  **Child**: 啟動，通知 Main，然後在其自己的條件變數上等待。
3.  **Main**: 醒來，繼續。

### 2.2 解鎖 (Unlocking)
它使用 `std::unique_lock` 和 `std::defer_lock` 來管理執行緒之間安全讓出控制權所需的精細鎖定順序。

---

## 3. 關鍵重點
1.  **現代替代方案**: 比原始 Pthreads 更乾淨，但具有相同的效能開銷 (作業系統層級切換)。
2.  **穩健性 (Robustness)**: 對於一般除錯或當 QuickThreads 在目標架構上不可用時，是較佳的選擇。
