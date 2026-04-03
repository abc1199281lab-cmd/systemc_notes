# SystemC 學習筆記

[English](README.md) | [繁體中文]

> **SystemC 函式庫的結構化學習筆記** — 從由上而下的概念架構到由下而上的原始碼解析。

## 目標讀者

沒有硬體 RTL 背景的軟體工程師，想要深入理解 SystemC。所有解釋力求連高中生都能理解。

## 文件目錄

| 目錄 | 語言 | 說明 |
|------|------|------|
| [zh-TW/](zh-TW/) | 繁體中文 | 概念、原始碼解析、範例、圖表 |
| [en/](en/) | English | 概念、原始碼解析、範例、圖表 |

### 內容結構

```
examples/    — SystemC 與 TLM 範例導讀（從這裡開始！）
topdown/     — 概念性架構文件（模擬引擎、事件、排程等）
code/        — 逐檔原始碼解析
diagrams/    — 架構總覽圖表
```

## 建議閱讀順序

1. **從範例開始** — 瀏覽 [examples/](zh-TW/examples/) 看 SystemC 實際怎麼用。多數開發者從範例學習最有效率
2. **需要時查閱概念** — 參考 [topdown/](zh-TW/topdown/) 了解設計背後的「為什麼」
3. **深入原始碼** — 探索 [code/](zh-TW/code/) 查看 SystemC 函式庫的逐檔解析
4. Mermaid 圖表可用 GitHub、VSCode 預覽或任何支援 Mermaid 的工具檢視

## 關於

本筆記用於系統性研讀 [Accellera SystemC](https://systemc.org/) 開源函式庫，涵蓋 SystemC 核心與 TLM（Transaction-Level Modeling）。
