# TLM LT 時間解耦 (Temporal Decoupling) 範例分析

> **來源**: `ref/systemc/examples/tlm/lt_temporal_decouple` & `ref/systemc/examples/tlm/common`

## 1. 概述
此範例展示了鬆散計時 (LT) 編碼風格中的 **時間解耦 (Temporal Decoupling)**。發起者被允許領先模擬時間一個 "量子 (Quantum)"，累加區域延遲而不是立即呼叫 `wait()`。這透過減少上下文切換顯著提高了模擬性能。

## 2. 關鍵組件

### 2.1 量子守護者 (`tlm_utils::tlm_quantumkeeper`)
-   **角色**: 管理發起者的區域時間偏移。
-   **全域量子**: 一個全域設定 (例如 500ns)，決定了發起者可以超前執行的程度。

### 2.2 TD 發起者 (`lt_td_initiator.cpp`)
-   **迴圈**:
    1.  **獲取偏移**: `m_delay = m_quantum_keeper.get_local_time()`。
    2.  **交辦交易**: 呼叫 `b_transport(trans, m_delay)`。目標將其處理時間加到 `m_delay` 中。
    3.  **更新守護者**: `m_quantum_keeper.set(m_delay)`。守護者現在知道新的區域時間。
    4.  **檢查同步**: `if (m_quantum_keeper.need_sync()) m_quantum_keeper.sync()`。
        -   如果累加的延遲 > 全域量子，守護者呼叫 `wait(delay)` 並將區域時間重設為 0。

### 2.3 同步目標 (`lt_synch_target.cpp`)
一個表現 "不規範" 或 "遺留" 的目標，它要求必須同步。
-   **行為**: 它不是僅僅疊加到 `delay_time`，而是明確地呼叫 `wait(delay_time)`。
-   **影響**: 它強迫模擬時間推進。
-   **重設**: 它充當一個同步點。在回傳前設定 `delay_time = SC_ZERO_TIME`。
-   **發起者處理**: 發起者看到的 `delay` 為 0。`m_quantum_keeper.set(0)` 正確地重設了區域偏移，因為目標已經 "支付" 了時間成本。

## 3. 執行流程
1.  **開始**: 全域量子 = 500ns。
2.  **交易 1**: 目標增加 100ns。`m_delay` = 100ns。不進行同步。
3.  **交易 2**: 目標增加 100ns。`m_delay` = 200ns。不進行同步。
...
4.  **交易 6**: 目標增加 100ns。`m_delay` = 600ns。
5.  **同步**: 600ns > 500ns。發起者呼叫 `wait(600ns)`。模擬時間推進。`m_delay` 重設為 0。

## 4. 關鍵重點
-   **速度**: 極大地減少了 `wait()` 呼叫和上下文切換的次數。
-   **權衡**: 降低了計時精確度。從其他行程的角度來看，在同一個量子內發生的事件實際上是 "同時" 發生的。
-   **最佳實踐**: 將時間解耦用於指令集模擬器 (ISS) 和高速功能模型。
