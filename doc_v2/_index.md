# SystemC 原始碼文件 v2

> 深入理解 SystemC 模擬框架的完整文件，涵蓋 top-down 架構概念與 bottom-up 逐檔解析。
> 適合軟體背景工程師，以國高中生也能理解的方式呈現。

## 導航

### Top-down 概念架構

以概念為主軸，由淺入深理解 SystemC 的設計與運作。

- [學習路徑](topdown/learning-path.md) - 建議閱讀順序與難度標示
- [模擬引擎](topdown/simulation-engine.md) - SystemC 核心引擎
- [事件機制](topdown/events.md) - 事件驅動模型
- [排程機制](topdown/scheduling.md) - process 排程與 delta cycle
- [模組層級](topdown/hierarchy.md) - module、port、channel 的組織結構
- [通訊機制](topdown/communication.md) - signal、port、channel、FIFO
- [資料型別](topdown/datatypes.md) - 硬體導向的資料型別
- [波形追蹤](topdown/tracing.md) - VCD/WIF 輸出
- [TLM](topdown/tlm.md) - Transaction Level Modeling

### Bottom-up 程式碼文件

逐檔解析 `ref/systemc/src/` 原始碼，保留完整目錄結構。

- [sysc/kernel/](code/sysc/kernel/_index.md) - 模擬核心引擎（69 個檔案）
- [sysc/communication/](code/sysc/communication/_index.md) - 通訊元件（39 個檔案）
- [sysc/datatypes/](code/sysc/datatypes/_index.md) - 資料型別（124 個檔案）
- [sysc/tracing/](code/sysc/tracing/_index.md) - 波形追蹤（9 個檔案）
- [sysc/utils/](code/sysc/utils/_index.md) - 工具函式庫（26 個檔案）
- [sysc/packages/](code/sysc/packages/_index.md) - 平台相關套件（Qt 協程）
- [tlm_core/](code/tlm_core/_index.md) - TLM 核心（47 個檔案）
- [tlm_utils/](code/tlm_utils/_index.md) - TLM 工具（16 個檔案）

### 全局圖表

- [SystemC 架構總覽](diagrams/systemc-overview.md) - 子系統關聯圖

## 對應規則

| 類型 | 原始碼路徑 | 文件路徑 |
|------|-----------|----------|
| Bottom-up | `ref/systemc/src/sysc/kernel/sc_event.cpp` | `doc_v2/code/sysc/kernel/sc_event.md` |
| Top-down | （概念主題） | `doc_v2/topdown/{topic}.md` |
