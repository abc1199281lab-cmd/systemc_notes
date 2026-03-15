## ADDED Requirements

### Requirement: doc_v2 目錄結構規劃

系統 SHALL 建立 `doc_v2/` 目錄，包含以下子目錄結構：
- `doc_v2/code/` — bottom-up 程式碼文件，鏡像 `ref/systemc/src/` 結構
- `doc_v2/topdown/` — top-down 概念架構文件
- `doc_v2/diagrams/` — 跨子系統的全局關聯圖（可選）

#### Scenario: 目錄結構建立

- **WHEN** 開始產出文件
- **THEN** SHALL 先建立完整的 `doc_v2/` 目錄骨架

#### Scenario: code 子目錄鏡像原始碼

- **WHEN** `ref/systemc/src/` 有子目錄 `sysc/kernel/`
- **THEN** `doc_v2/code/` SHALL 有對應子目錄 `sysc/kernel/`

### Requirement: 命名規則一致性

所有文件 SHALL 遵循以下命名規則：
- 檔名使用英文小寫與連字號（kebab-case）
- Bottom-up：原始碼檔名去掉副檔名加 `.md`（如 `sc_event.md`）
- Top-down：概念名稱加 `.md`（如 `simulation-engine.md`）

#### Scenario: bottom-up 命名

- **WHEN** 原始碼檔案為 `sc_event.cpp`
- **THEN** 文件命名 SHALL 為 `sc_event.md`

#### Scenario: top-down 命名

- **WHEN** 主題為 simulation engine
- **THEN** 文件命名 SHALL 為 `simulation-engine.md`

### Requirement: 索引與導航機制

每個子目錄 SHALL 包含 `_index.md` 索引頁面，列出該目錄下所有文件並附簡要說明，提供快速導航。

#### Scenario: 子目錄索引

- **WHEN** 使用者開啟 `doc_v2/code/sysc/kernel/`
- **THEN** SHALL 有 `_index.md` 列出所有 kernel 相關文件及其一行說明

#### Scenario: 頂層索引

- **WHEN** 使用者開啟 `doc_v2/`
- **THEN** SHALL 有 `_index.md` 提供整體導航，連結至 `code/` 與 `topdown/`

### Requirement: 分 Phase 交付

文件產出 SHALL 按 phase 分批進行，每完成一個 phase 即 commit 並 push。Phase 順序依照子系統重要性排列。

#### Scenario: Phase 劃分

- **WHEN** 規劃工作順序
- **THEN** SHALL 至少包含以下 phase：P1-kernel、P2-communication、P3-datatypes、P4-tracing、P5-utils、P6-tlm、P7-topdown

#### Scenario: 每 phase commit

- **WHEN** 完成一個 phase 的所有文件
- **THEN** SHALL 執行 git commit 並 push 至 remote
