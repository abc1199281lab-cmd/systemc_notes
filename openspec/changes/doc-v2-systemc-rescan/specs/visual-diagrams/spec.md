## ADDED Requirements

### Requirement: Mermaid 圖表嵌入文件

所有 `doc_v2/` 中的文件 SHALL 在適當處嵌入 Mermaid 圖表，使用 Markdown fenced code block 語法（```mermaid）。

#### Scenario: 圖表格式

- **WHEN** 文件需要展示流程或關係
- **THEN** SHALL 使用 Mermaid 語法嵌入，而非外部圖片檔案

#### Scenario: 渲染相容性

- **WHEN** 使用者在 GitHub 或 Obsidian 開啟文件
- **THEN** Mermaid 圖表 SHALL 正確渲染

### Requirement: 支援多種圖表類型

文件 SHALL 根據內容需要使用不同類型的 Mermaid 圖表：類別圖（classDiagram）、序列圖（sequenceDiagram）、流程圖（flowchart）、狀態圖（stateDiagram）。

#### Scenario: 類別關係使用類別圖

- **WHEN** 說明 class 之間的繼承或組合關係
- **THEN** SHALL 使用 Mermaid `classDiagram`

#### Scenario: 時序互動使用序列圖

- **WHEN** 說明模組間的呼叫順序（如 simulation kernel 的 evaluate-update cycle）
- **THEN** SHALL 使用 Mermaid `sequenceDiagram`

#### Scenario: 流程說明使用流程圖

- **WHEN** 說明程式執行流程或決策邏輯
- **THEN** SHALL 使用 Mermaid `flowchart`

### Requirement: 跨檔案依賴關係圖

每個子系統（如 `sysc/kernel/`）SHALL 包含一張總覽依賴關係圖，展示該目錄下檔案間的 include/呼叫關係。

#### Scenario: 子系統依賴總覽

- **WHEN** 使用者開啟 `doc_v2/code/sysc/kernel/` 目錄的索引文件
- **THEN** SHALL 看到一張 Mermaid 圖表展示 kernel 目錄下各檔案的依賴關係

#### Scenario: 圖表節點限制

- **WHEN** 某子系統包含超過 15 個相關檔案
- **THEN** SHALL 將依賴關係拆分為多張圖表，每張不超過 15 個節點
