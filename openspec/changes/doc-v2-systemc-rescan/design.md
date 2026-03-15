## Context

目前 `doc/formal/` 中的文件由 AI 生成，但顆粒度不足，無法一對一對應 `ref/systemc/src/` 中約 246 個原始碼檔案。現有的 `doc/code/` 已有部分底層文件，`doc/topdown/` 有概念架構文件，`doc/zh_tw/` 有繁體中文翻譯，但整體缺乏：
- 與原始碼的精確對應
- UML/依賴關係等視覺化圖表
- 跨檔案關聯說明
- 適合初學者的學習路徑

新版文件將放在 `doc_v2/`，不影響現有 `doc/` 目錄。

## Goals / Non-Goals

**Goals:**

- 為 `ref/systemc/src/` 中每個原始碼檔案產出對應的 bottom-up 文件
- 建立以概念為主軸的 top-down 學習路徑
- 所有文件嵌入 Mermaid 圖表（序列圖、類別圖、依賴關係圖）
- 內容以繁體中文撰寫，適合國高中生理解
- 補充硬體 RTL 背景知識，解釋「為什麼要這樣設計」
- 支援 GitHub 與 Obsidian 渲染
- 每完成一個 phase 即 commit 並 push

**Non-Goals:**

- 不修改或取代現有 `doc/` 目錄的內容
- 不修改 `ref/systemc/` 中的任何原始碼
- 不產出英文版文件（本次僅繁體中文）
- 不做互動式教學工具或網站

## Decisions

### 決策 1：使用 `doc_v2/` 而非修改現有 `doc/`

**選擇**：建立全新 `doc_v2/` 目錄
**理由**：現有 `doc/` 有大量已審閱的內容與翻譯，直接修改風險高。獨立目錄可以平行開發，完成後再決定是否取代。
**替代方案**：在 `doc/` 下建立子目錄（如 `doc/v2/`）——但路徑太深，不利導航。

### 決策 2：Bottom-up 文件保留完整目錄結構

**選擇**：`ref/systemc/src/sysc/kernel/sc_event.cpp` → `doc_v2/code/sysc/kernel/sc_event.md`
**理由**：一對一對應讓使用者能快速從原始碼找到文件，反之亦然。
**替代方案**：按功能重新分類——但會失去與原始碼的直覺對應。

### 決策 3：Top-down 採用分層學習路徑

**選擇**：以 `doc_v2/topdown/` 存放概念文件，搭配 `learning-path.md` 索引
**理由**：初學者需要由淺入深的引導，不能只看單一檔案的說明。學習路徑提供閱讀順序建議。
**替代方案**：全部放在 README 中——但內容太多不適合單一檔案。

### 決策 4：Mermaid 作為圖表格式

**選擇**：使用 Mermaid 語法嵌入 Markdown
**理由**：GitHub 原生支援、Obsidian 相容、不需要額外工具或圖片檔案、可版本控制。
**替代方案**：PlantUML（需外部渲染器）、手繪圖片（無法版本控制）。

### 決策 5：分 Phase 執行，每 phase commit + push

**選擇**：按 SystemC 子系統分 phase（kernel → communication → datatypes → tracing → tlm）
**理由**：每個子系統相對獨立，可以逐步交付並檢視品質。與 `ref/systemc/src/sysc/` 的目錄結構自然對應。
**替代方案**：按檔案字母順序——但失去邏輯連貫性。

## Risks / Trade-offs

- **[原始碼數量龐大]** 約 246 個檔案，工作量大 → 分 phase 執行，優先處理核心子系統（kernel, communication）
- **[Mermaid 圖表複雜度]** 過於複雜的圖表在 GitHub 上渲染慢或不清晰 → 控制每張圖的節點數在 15 以內，複雜關係拆分多張圖
- **[內容深度與易懂性平衡]** 要兼顧技術深度與國高中生理解 → 每個概念先用類比說明，再深入技術細節
- **[symlink 依賴]** `ref/systemc/` 是 symlink，CI 環境可能無法存取 → 文件產出後為獨立 Markdown，不依賴原始碼即可閱讀
