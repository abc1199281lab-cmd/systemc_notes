## 1. 初始設定與目錄結構

- [x] 1.1 建立 `doc_v2/` 目錄骨架（`code/`, `topdown/`, `diagrams/`）
- [x] 1.2 建立 `doc_v2/_index.md` 頂層索引頁面
- [x] 1.3 掃描 `ref/systemc/src/` 取得完整檔案清單，規劃對應的 `doc_v2/code/` 目錄結構

## 2. P1 - Kernel 子系統（Bottom-up）

- [x] 2.1 建立 `doc_v2/code/sysc/kernel/` 目錄與 `_index.md`
- [x] 2.2 為 `ref/systemc/src/sysc/kernel/` 下每個 `.cpp`/`.h` 檔案產出對應文件（含 class/function 說明、設計原理、RTL 背景）
- [x] 2.3 為 kernel 子系統建立依賴關係圖（Mermaid classDiagram / flowchart）
- [x] 2.4 Commit 並 push P1

## 3. P2 - Communication 子系統（Bottom-up）

- [x] 3.1 建立 `doc_v2/code/sysc/communication/` 目錄與 `_index.md`
- [x] 3.2 為 `ref/systemc/src/sysc/communication/` 下每個檔案產出對應文件
- [x] 3.3 為 communication 子系統建立依賴關係圖
- [x] 3.4 Commit 並 push P2

## 4. P3 - Datatypes 子系統（Bottom-up）

- [x] 4.1 建立 `doc_v2/code/sysc/datatypes/` 目錄與 `_index.md`
- [x] 4.2 為 `ref/systemc/src/sysc/datatypes/` 下每個檔案產出對應文件
- [x] 4.3 為 datatypes 子系統建立依賴關係圖
- [x] 4.4 Commit 並 push P3

## 5. P4 - Tracing 子系統（Bottom-up）

- [x] 5.1 建立 `doc_v2/code/sysc/tracing/` 目錄與 `_index.md`
- [x] 5.2 為 `ref/systemc/src/sysc/tracing/` 下每個檔案產出對應文件
- [x] 5.3 為 tracing 子系統建立依賴關係圖
- [x] 5.4 Commit 並 push P4

## 6. P5 - Utils 與 Packages（Bottom-up）

- [x] 6.1 建立 `doc_v2/code/sysc/utils/` 與 `doc_v2/code/sysc/packages/` 目錄與 `_index.md`
- [x] 6.2 為 utils 與 packages 下每個檔案產出對應文件
- [x] 6.3 Commit 並 push P5

## 7. P6 - TLM（Bottom-up）

- [x] 7.1 建立 `doc_v2/code/tlm_core/` 與 `doc_v2/code/tlm_utils/` 目錄與 `_index.md`
- [x] 7.2 為 `ref/systemc/src/tlm_core/` 與 `tlm_utils/` 下每個檔案產出對應文件
- [x] 7.3 為 TLM 子系統建立依賴關係圖
- [x] 7.4 Commit 並 push P6

## 8. P7 - Top-down 概念架構文件

- [x] 8.1 建立 `doc_v2/topdown/` 目錄與 `_index.md`
- [x] 8.2 撰寫 `learning-path.md`（學習路徑索引，含難度標示與前置知識）
- [x] 8.3 撰寫 `simulation-engine.md`（模擬引擎核心概念）
- [x] 8.4 撰寫 `events.md`（事件機制）
- [x] 8.5 撰寫 `scheduling.md`（排程機制）
- [x] 8.6 撰寫 `hierarchy.md`（模組層級結構）
- [x] 8.7 撰寫 `communication.md`（通訊機制：port, channel, signal）
- [x] 8.8 撰寫 `datatypes.md`（資料型別）
- [x] 8.9 撰寫 `tracing.md`（波形追蹤）
- [x] 8.10 撰寫 `tlm.md`（Transaction Level Modeling）
- [x] 8.11 每份 top-down 文件加入「相關模組」段落與跨模組 Mermaid 圖表
- [x] 8.12 Commit 並 push P7

## 9. 收尾與全局整合

- [x] 9.1 更新 `doc_v2/_index.md` 頂層索引，確保所有文件均有連結
- [x] 9.2 建立 `doc_v2/diagrams/` 全局架構總覽圖（SystemC 整體子系統關聯）
- [x] 9.3 最終檢查：確認所有 Mermaid 圖表可在 GitHub 渲染、所有連結有效
- [x] 9.4 Final commit 並 push
