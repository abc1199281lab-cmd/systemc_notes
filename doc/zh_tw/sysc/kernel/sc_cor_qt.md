# SystemC Coroutine (QuickThreads) 分析

> **檔案**: `ref/systemc/src/sysc/kernel/sc_cor_qt.cpp`

## 1. 概述
此檔案使用 **QuickThreads** (QT) 實作 SystemC 協程介面。這通常是 Unix-like 系統上的預設且效能最好的後端，因為它完全在使用者空間執行上下文切換。

## 2. 核心機制

### 2.1 堆疊管理 (Stack Management)
- **`stack_alloc`**: 手動 (透過 `mmap`) 為每個協程/執行緒分配記憶體作為堆疊。
- **`stack_protect`**: 使用 `mprotect` 在堆疊末端增加一個 "警戒區 (red zone)" 以偵測溢位。

### 2.2 上下文切換 (`yield`)
- 使用組合語言輔助函式 (`QUICKTHREADS_BLOCK`, `sc_cor_qt_yieldhelp`) 重寫 CPU 堆疊指標 (`SP`) 和程式計數器 (`PC`)。
- **Sanitizers**: 包含掛鉤 (`sanitizer_start_switch_fiber`) 以配合 AddressSanitizer (ASan)，確保它知道堆疊切換。

---

## 3. 關鍵重點
1.  **速度**: 比 Pthreads 快，因為它避免了切換時的核心系統呼叫 (syscalls)。
2.  **脆弱性 (Fragility)**: 依賴特定架構的組合語言程式碼 (通常在連結進來的單獨 `.s` 檔案中) 來捕捉和恢復暫存器狀態。
