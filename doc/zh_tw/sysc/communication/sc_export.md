# SystemC Export 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_export.cpp`

## 1. 概述
`sc_export` 允許模組向外部 **提供 (provide)** 一個介面 (例如，暴露內部通道或 `this` 模組作為介面的實作)。

## 2. 與 Ports 的相似之處
- 技術上與 `sc_port` 非常相似。
- 它有一個註冊表 (`sc_export_registry`)。
- 它可以被綁定。

## 3. 差異
- **方向 (Direction)**: Port "使用" 介面。Export "提供" 介面。
- **綁定 (Binding)**: 當你綁定 `port.bind(export)` 時，port 獲得 export 所持有的介面指標。
- **階層 (Hierarchy)**: Exports 允許將詳細的內部實作 "向上" 暴露給更高層次的抽象。

## 4. 生命週期
像 ports 一樣，exports 有生命週期回調 (`before_end_of_elaboration`, `start_of_simulation` 等)，由註冊表管理。

---

## 5. 關鍵重點
1.  **結構連接性 (Structural Connectivity)**: Exports 對於階層化設計至關重要，其中父模組包裝一個通道並透過 export 將其暴露給兄弟模組。
