# TLM 全域量子 (Global Quantum) 分析

> **檔案**: `ref/systemc/src/tlm_core/tlm_2/tlm_quantum/tlm_global_quantum.h`

## 1. 概述
`tlm_global_quantum` 類別管理模擬的全域時間量子 (global time quantum)。它是一個單例 (singleton)。

## 2. 概念：時間解耦 (Temporal Decoupling)
在鬆散計時 (Loosely Timed, LT) 建模中，發起者 (initiators) 被允許超前於全域 SystemC 模擬時間 (`sc_time_stamp()`) 執行，藉由減少上下文切換 (呼叫 `wait()`) 的次數來提高效能。
**全域量子 (Global Quantum)** 定義了發起者在 *必須* 同步之前可以領先執行的最大時間量。

## 3. 實作
- **單例**: 透過 `instance()` 存取。
- **`m_global_quantum`**: 儲存量子的時間持續長度。
- **`compute_local_quantum()`**: 回傳全域量子。(雖然在更複雜的設定中，衍生類別可能會覆寫此邏輯，但通常它只是回傳全域值)。

## 4. 關鍵重點
1.  **效能對比精確度**: 較大的量子允許更快的模擬 (同步次數較少)，但會降低時序精確度和行程間的互動性。較小的量子會增加精確度，但會降低模擬速度。
