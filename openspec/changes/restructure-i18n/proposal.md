## Why

目前 `doc/`（舊版）和 `doc_v2/`（新版繁中）並存，結構混亂且缺少英文版本。需要統一目錄結構、移除舊版、建立多語系架構，讓中英文讀者都有清楚的入口。

## What Changes

- **BREAKING** 刪除 `doc/` 目錄（舊版，178 檔，已被 doc_v2 取代）
- `doc_v2/` 重新命名為 `zh-TW/`（繁體中文筆記）
- 新增 `en/` 目錄，翻譯 `zh-TW/` 的 `topdown/`、`examples/`、`diagrams/` 為英文
- 新增 `README.md`（English）與 `README_zh-TW.md`（繁體中文），互相連結
- 更新 `AGENTS.md` 的 folder structure 段落

## Capabilities

### New Capabilities
- `english-docs`: 將 zh-TW 的 topdown/、examples/、diagrams/ 翻譯成英文，鏡像結構放在 en/
- `bilingual-readme`: 新增雙語 README（en + zh-TW），包含專案介紹與目錄導航

### Modified Capabilities

（無既有 spec 需要修改）

## Impact

- 所有 `doc/` 內的連結與引用將失效（已被 doc_v2 取代，影響不大）
- `doc_v2/` 路徑變更為 `zh-TW/`，需更新任何引用此路徑的地方
- `AGENTS.md` 的 folder structure 需同步更新
