# TLM-2.0 Core: Generic Payload 實作 (`tlm_gp`)

> **來源**: `ref/systemc/src/tlm_core/tlm_2/tlm_generic_payload/tlm_gp.cpp`

## 1. 概述
此檔案實作了 `tlm_generic_payload` 類別及其底層的擴充機制 (extension mechanism)。它處理擴充功能的動態註冊以及交易 (transactions) 的深層複製 (deep copy) 邏輯。

## 2. 擴充機制 (`tlm_extension_registry`)
-   **單例註冊表 (Singleton Registry)**: `tlm_extension_registry` 將 `std::type_index` (來自 `typeid`) 映射到唯一的整數 ID (`unsigned int`)。
-   **註冊**: 如果該型別之前未出現過，`register_extension` 會分配一個新 ID。
-   **用法**: `tlm_extension_base` 使用此註冊表來獲取衍生擴充類別的 ID。

## 3. Generic Payload 實作
-   **建構**: 將所有欄位 (地址、命令、資料指標等) 初始化為預設值。
-   **深層複製 (Deep Copy)** (`deep_copy_from`):
    -   複製標準屬性 (地址、命令等)。
    -   **資料**: 如果存在資料緩衝區，則進行 `memcpy`。
    -   **位元組啟用 (Byte Enables)**: 如果存在位元組啟用緩衝區，則進行 `memcpy`。
    -   **擴充功能 (Extensions)**: 遍歷所有擴充功能。
        -   如果原始物件有擴充功能 `i` 而我們沒有：呼叫 `clone()`。
        -   如果兩者都有擴充功能 `i`：呼叫 `copy_from()`。
-   **更新原始物件 (Update Original)** (`update_original_from`):
    -   用於 `transport_dbg` 或傳輸的回傳路徑。
    -   複製回擴充功能、回應狀態 (response status)、DMI 提示。
    -   **讀取資料**: 如果命令是 READ，則將資料從副本複製回原始物件，並尊重位元組啟用 (針對 4/8 位元組字元進行了最佳化)。
-   **記憶體管理**:
    -   `free_all_extensions()`: 明確釋放所有擴充功能。
    -   解構子: 釋放仍然掛載的擴充功能。

## 4. 關鍵輔助工具
-   `set_extension`, `get_extension`, `clear_extension`: 使用擴充功能陣列的存取器。
-   `resize_extensions`: 如果全域註冊了新的擴充功能，則將擴充功能陣列擴大到 `max_num_extensions()`。
