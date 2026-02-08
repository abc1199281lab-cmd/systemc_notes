# SystemC Coroutine (Windows Fibers) 分析

> **檔案**: `ref/systemc/src/sysc/kernel/sc_cor_fiber.cpp`

## 1. 概述
此檔案使用 **Windows Fibers** 實作 SystemC 協程 (coroutines)。Fibers 是 Windows API 提供的輕量級類執行緒 (thread-like) 原語，專門用於協作式多工 (cooperative multitasking)。

## 2. 核心機制

### 2.1 主執行緒轉換 (Main Thread Conversion)
- **`ConvertThreadToFiber`**: 主模擬執行緒必須先轉換為 Fiber，然後才能排程其他 Fibers。

### 2.2 建立與切換 (Creation and Switching)
- **`CreateFiberEx`**: 建立由作業系統管理的協程堆疊管理結構。
- **`SwitchToFiber`**: 實際的上下文切換呼叫。它儲存當前狀態並恢復目標 fiber 狀態。

---

## 3. 關鍵重點
1.  **OS 支援**: 這是 Windows 上實作使用者空間執行緒 (user-space threads) 的原生方式。
2.  **SJLJ 例外**: 在某些 GCC/MinGW 設定上，它包含特殊的處理 (`_Unwind_SjLj_Register`) 以確保 C++ 例外能正確地跨越 fiber 切換傳播。
