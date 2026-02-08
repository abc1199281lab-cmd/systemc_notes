# SystemC Exception 分析

> **檔案**: `ref/systemc/src/sysc/kernel/sc_except.cpp`

## 1. 概述
此檔案處理模擬核心內的例外 (exception) 轉換和處理。它特別定義了 `sc_unwind_exception`，用於實作 `sc_thread` 終止 (重置或刪除)。

## 2. Unwinding (`sc_unwind_exception`)
這是核心丟入 thread 中的一個特殊例外，以強制它終止或重置。
- **`is_reset`**: 區分 "Kill" (終止) 和 "Reset" (重啟)。
- **`what()`**: 返回 "KILL" 或 "RESET"。
- **安全性**: 解構子檢查它是否在仍然活動時被銷毀 (例如，如果使用者試圖吞下它)。如果是這樣，它會發出致命錯誤，因為 unwinding 不能被阻擋。

## 3. 全域例外處理器 (`sc_handle_exception`)
由 `sc_main` 和核心的行程調度器使用的全域捕捉函式，用於捕捉從使用者程式碼逃逸的例外。
- 捕捉 `sc_report`
- 捕捉 `std::exception`
- 捕捉未知例外 (`...`)
- 將它們全部包裝成動態分配的 `sc_report*` 以進行統一處理。

---

## 4. 關鍵重點
1.  **不要吞下 Unwind (Don't Swallow Unwind)**: 使用者程式碼通常不應該捕捉 `sc_unwind_exception` (或 `...`) 而不重新丟出，否則，thread/process 無法正確關閉或重置。
