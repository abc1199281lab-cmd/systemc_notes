## Why

現有 `doc/formal/` 中的文件顆粒度不足，無法一對一對應 `ref/systemc/src/` 的原始碼結構。缺乏 UML 序列圖、依賴關係圖等視覺化輔助，也沒有清楚的跨檔案關聯說明。需要重新掃描所有原始碼，產出結構化、易懂、且適合國高中生理解的第二版文件（`doc_v2/`），同時加入學習路徑與 Mermaid 圖表。

## What Changes

- 全面重新掃描 `ref/systemc/src/` 下所有原始碼，產出 `doc_v2/` 第二版文件
- Bottom-up 文件：`ref/systemc/src/sysc/kernel/sc_event.cpp` → `doc_v2/code/sysc/kernel/sc_event.md`，保留完整目錄結構，每個檔案對應一份文件
- Top-down 文件：`doc_v2/topdown/{topic}.md`，以概念為主軸的架構學習路徑
- 加入 Mermaid 圖表（UML 序列圖、依賴關係圖、流程圖）
- 加入不同類別/檔案間的關聯圖
- 加入學習路徑（learning path）指引
- 繁體中文內容，英文檔名/程式碼/專有名詞
- 參考 [repo-lantern](https://github.com/abc1199281/repo-lantern) 的規劃思路
- 每完成一個 phase 即 commit 並 push

## Capabilities

### New Capabilities
- `bottom-up-code-docs`: 逐檔掃描 `ref/systemc/src/` 產出與原始碼目錄結構一對一對應的 bottom-up 程式碼文件，涵蓋 class/function 說明、設計原理、硬體 RTL 背景知識
- `topdown-learning-path`: 以概念為主軸的 top-down 架構文件，包含學習路徑規劃與跨模組關聯說明
- `visual-diagrams`: 在文件中嵌入 Mermaid 圖表（UML 序列圖、依賴關係、元件互動流程圖、跨檔案關聯圖）
- `doc-v2-structure`: `doc_v2/` 整體目錄結構規劃、命名規則、索引頁面與導航機制

### Modified Capabilities

（無既有 spec 需修改）

## Impact

- 新增 `doc_v2/` 目錄，包含 `code/` 與 `topdown/` 子目錄
- 預計產出數十至上百個 Markdown 檔案，對應 `ref/systemc/src/` 中約 246 個原始碼檔案
- 不影響現有 `doc/` 目錄內容
- 需要存取 `ref/systemc/src/` 目錄（symlink 至 `/home/powei/MyLearning/systemc`）
- 使用 Mermaid 語法，相容 GitHub 與 Obsidian 渲染
