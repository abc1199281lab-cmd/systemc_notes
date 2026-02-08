# TLM-2.0 Core: Phase 實作 (`tlm_phase`)

> **來源**: `ref/systemc/src/tlm_core/tlm_2/tlm_generic_payload/tlm_phase.cpp`

## 1. 概述
此檔案實作了 `tlm_phase`，該類別代表交易的階段 (例如 `BEGIN_REQ`, `END_RESP`)。它使用與擴充功能類似的註冊表模式，以支援使用者自定義的階段。

## 2. 架構 (`tlm_phase_registry`)
-   **單例 (Singleton)**: `tlm_phase_registry` 管理階段型別與 ID 之間的映射。
-   **儲存**:
    -   `ids_`: `std::type_index` 到 `unsigned int` (ID) 的映射。
    -   `names_`: 字串向量，由 ID 索引。
-   **內建階段**: 建構子預先註冊了標準階段：
    -   `UNINITIALIZED_PHASE`
    -   `BEGIN_REQ`, `END_REQ`
    -   `BEGIN_RESP`, `END_RESP`

## 3. 功能
-   **註冊**: `register_phase` 確保每個 `std::type_info` 都有唯一的 ID。透過一次性初始化，它是執行緒安全的。
-   **名稱檢索**: `get_name()` 從註冊表中回傳階段的字串表示形式。

## 4. 用法
使用者通常不會直接與註冊表互動。他們實例化 `tlm_phase` (或衍生類別/常數)，這會在建構時於內部呼叫 `register_phase`。
