## ADDED Requirements

### Requirement: 每個原始碼檔案對應一份文件

系統 SHALL 為 `ref/systemc/src/` 下每個 `.cpp` 和 `.h` 檔案產出對應的 Markdown 文件，存放於 `doc_v2/code/` 並保留完整目錄結構。

#### Scenario: 產出對應文件

- **WHEN** 原始碼路徑為 `ref/systemc/src/sysc/kernel/sc_event.cpp`
- **THEN** 產出文件路徑 SHALL 為 `doc_v2/code/sysc/kernel/sc_event.md`

#### Scenario: 目錄結構完整保留

- **WHEN** 原始碼位於子目錄 `ref/systemc/src/sysc/communication/`
- **THEN** 文件 SHALL 位於對應子目錄 `doc_v2/code/sysc/communication/`

### Requirement: 文件內容涵蓋 class 與 function 說明

每份 bottom-up 文件 SHALL 包含該檔案中所有 class、struct、function 的說明，包括：用途、參數、回傳值、與其他元件的關係。

#### Scenario: class 說明完整

- **WHEN** 原始碼檔案包含 `class sc_event`
- **THEN** 文件 SHALL 包含該 class 的用途說明、成員函式列表、繼承關係、使用範例

#### Scenario: 多個 class 的檔案

- **WHEN** 原始碼檔案包含多個 class 或 function
- **THEN** 文件 SHALL 逐一說明每個 class/function，以標題區分

### Requirement: 說明設計原理與硬體 RTL 背景

每份文件 SHALL 解釋「為什麼要這樣設計」，並在適當處補充硬體 RTL 的對應概念。

#### Scenario: 設計原理說明

- **WHEN** 原始碼使用特定設計模式（如 observer pattern）
- **THEN** 文件 SHALL 說明為何採用該模式，以及在硬體模擬中的意義

#### Scenario: RTL 背景補充

- **WHEN** 程式碼概念對應硬體概念（如 `sc_signal` 對應硬體 wire/register）
- **THEN** 文件 SHALL 補充硬體 RTL 的類比說明

### Requirement: 內容以繁體中文撰寫且適合國高中生理解

所有文件內容 SHALL 以繁體中文撰寫，專有名詞與程式碼使用英文。說明方式 SHALL 先用生活化類比，再深入技術細節。

#### Scenario: 繁體中文內容

- **WHEN** 產出任一份文件
- **THEN** 說明文字 SHALL 為繁體中文，檔名與程式碼引用 SHALL 為英文

#### Scenario: 易懂性

- **WHEN** 說明一個複雜概念（如 delta cycle）
- **THEN** 文件 SHALL 先用日常生活類比解釋，再給出技術定義
