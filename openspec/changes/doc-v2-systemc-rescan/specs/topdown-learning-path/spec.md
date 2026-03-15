## ADDED Requirements

### Requirement: 概念為主軸的 top-down 架構文件

系統 SHALL 在 `doc_v2/topdown/` 產出以概念為主軸的架構文件，涵蓋 SystemC 的核心子系統：simulation engine、events、scheduling、hierarchy、communication、datatypes、tracing、TLM。

#### Scenario: 核心子系統文件

- **WHEN** 產出 top-down 文件
- **THEN** SHALL 至少包含以下主題檔案：`simulation-engine.md`、`events.md`、`scheduling.md`、`hierarchy.md`、`communication.md`、`datatypes.md`、`tracing.md`、`tlm.md`

#### Scenario: 概念說明而非逐檔說明

- **WHEN** 使用者閱讀 top-down 文件
- **THEN** 文件 SHALL 以概念為單位組織內容，跨檔案整合相關知識，而非逐一列舉程式碼檔案

### Requirement: 學習路徑索引

系統 SHALL 提供 `doc_v2/topdown/learning-path.md` 作為學習路徑索引，建議閱讀順序，標示難度等級，並說明各主題的前置知識。

#### Scenario: 閱讀順序建議

- **WHEN** 使用者開啟 `learning-path.md`
- **THEN** SHALL 看到由淺入深的閱讀順序，標示每個主題的難度（入門/進階/深入）

#### Scenario: 前置知識標示

- **WHEN** 某主題需要先理解其他主題
- **THEN** `learning-path.md` SHALL 標示該主題的前置知識（如「閱讀 communication 前請先理解 events」）

### Requirement: 跨模組關聯說明

每份 top-down 文件 SHALL 包含「相關模組」段落，說明與其他子系統的關聯與互動方式。

#### Scenario: 關聯說明

- **WHEN** 使用者閱讀 `events.md`
- **THEN** SHALL 看到 events 與 scheduling、communication 等模組的關聯說明

#### Scenario: 跨模組互動圖

- **WHEN** 多個模組有互動關係
- **THEN** 文件 SHALL 包含 Mermaid 圖表展示模組間的互動流程
