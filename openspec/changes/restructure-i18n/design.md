## Context

systemc_notes 目前有兩套文件目錄：`doc/`（舊版，英文為主 + zh_tw 子目錄）和 `doc_v2/`（新版，全繁中，316 檔）。`doc_v2` 已完全取代 `doc`，但命名和結構未反映多語系需求。需要重組為語言分層目錄，並新增英文翻譯。

## Goals / Non-Goals

**Goals:**
- 統一為語言做頂層目錄的架構（`zh-TW/`, `en/`）
- 移除已棄用的 `doc/` 目錄
- 翻譯 topdown/、examples/、diagrams/ 為英文
- 提供雙語 README 作為專案入口

**Non-Goals:**
- 翻譯 `code/` 目錄（量太大，未來再做）
- 修改文件內容本身（只翻譯，不改寫）
- 建立自動化翻譯 pipeline

## Decisions

### 1. 語言做頂層目錄（方案 A）

選擇 `zh-TW/` + `en/` 做頂層目錄，而非 `docs/zh-TW/` + `docs/en/`。

**理由**：這個 repo 本身就是文件，不需要再包一層 `docs/`。頂層直接分語言最簡潔。

### 2. `en/` 只翻譯三個子目錄

`en/` 鏡像 `zh-TW/` 的 `topdown/`、`examples/`、`diagrams/`，暫不翻譯 `code/`。

**理由**：`code/` 有 200+ 檔案，翻譯量太大。topdown + examples + diagrams 是最高價值的概念文件，先做這些。

### 3. git mv 保留歷史

使用 `git mv doc_v2 zh-TW` 而非刪除再建立，保留 git 歷史追蹤。

### 4. README 風格

參考 repo-lantern 的雙語 README 模式：`README.md`（English）+ `README_zh-TW.md`（繁中），互相在頂部連結。內容以「專案介紹 + 文件導航 + 建議閱讀順序」為主。

## Risks / Trade-offs

- **路徑變更破壞引用** → `doc_v2/` 只在 AGENTS.md 和 openspec 內被引用，影響範圍小。`doc/` 已無外部引用。git mv 後 GitHub 上的舊連結會 404，但這是學習筆記 repo，影響可接受。
- **翻譯品質** → AI 翻譯可能不夠精確，尤其是 SystemC 專有名詞。Mitigation：保持技術術語英文不譯，翻譯後人工 review。
- **en/ 與 zh-TW/ 內容不同步** → 未來 zh-TW 更新時 en/ 可能落後。Mitigation：目前可接受，未來若需要再建立同步機制。
