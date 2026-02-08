# SystemC Version 分析

> **檔案**: `ref/systemc/src/sysc/kernel/sc_ver.cpp`

## 1. 概述
此檔案包含全域版本字串、版權訊息和驗證檢查的定義。

## 2. 版本資訊
- 定義 `systemc_version` 字串 (例如 "SystemC 2.3.4 ...")。
- 定義組件: `sc_version_major`, `sc_version_minor`, `sc_version_patch`。
- `sc_copyright()`: 返回標準 Accellera 版權字串。

## 3. 連結時/執行時檢查 (Link-Time/Run-Time Checks)
- **`SC_API_PERFORM_CHECK_`**: 用於確保應用程式和函式庫是使用一致的設定 (例如 `SC_DEFAULT_WRITER_POLICY`) 編譯的巨集。
- 如果偵測到不匹配 (例如應用程式使用一種策略，函式庫使用另一種)，則報告致命錯誤以防止微妙的 ABI bugs。

---

## 4. 關鍵重點
1.  **ABI 安全**: `sc_api_version_...` 建構子邏輯在程式啟動期間 (靜態初始化時間) 執行關鍵的健全性檢查 (sanity checks)，以確保二進位相容性。
