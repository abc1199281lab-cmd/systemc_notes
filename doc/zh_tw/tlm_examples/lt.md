# TLM 鬆散計時 (Loosely Timed, LT) 範例分析

> **來源**: `ref/systemc/examples/tlm/lt` & `ref/systemc/examples/tlm/common`

## 1. 概述
此範例展示了 TLM 2.0 中的 **鬆散計時 (Loosely Timed, LT)** 編碼風格。LT 風格用於高性能軟體開發 (虛擬原型製作)，在其中用計時精確度來換取模擬速度。

## 2. 關鍵特性
1.  **阻塞傳輸 (Blocking Transport)**: 發起者 (initiator) 使用 `b_transport`。
2.  **時間解耦 (Temporal Decoupling)**: 發起者累加區域時間延遲，僅在必要時 (或在交易結束後) 才呼叫 `wait()`。
3.  **無階段 (No Phases)**: 交易在單次函式呼叫中完成。

## 3. 實作細節

### 3.1 發起者 (`lt_initiator.cpp`)
發起者在 `SC_THREAD` 中執行。
-   **機制**:
    ```cpp
    // 1. 獲取 generic payload
    transaction_ptr = request_in_port->read();

    // 2. 呼叫阻塞傳輸 (此時尚未消耗模擬時間)
    sc_time delay = SC_ZERO_TIME;
    initiator_socket->b_transport(*transaction_ptr, delay);

    // 3. 同步 (消耗時間)
    // 目標將其處理時間加到了 'delay' 中。
    // 發起者負責呼叫 wait()。
    wait(delay); 
    ```
-   **通訊端 (Socket)**: 使用 `tlm_utils::simple_initiator_socket`。

### 3.2 目標 (`lt_target.cpp`)
目標實作傳輸邏輯。
-   **機制**:
    ```cpp
    void lt_target::custom_b_transport(tlm_generic_payload &payload, sc_time &delay_time)
    {
        // 1. 執行操作 (讀取/寫入記憶體)
        m_target_memory.operation(payload, mem_op_time);

        // 2. 標註計時 (不要呼叫 wait)
        delay_time += mem_op_time + m_accept_delay;
    }
    ```
-   **通訊端**: 使用 `tlm_utils::simple_target_socket` 並註冊 `custom_b_transport` 作為回調。

## 4. 執行流程
1.  **開始**: 發起者在 T=0 時發送 `b_transport`。
2.  **目標**: 瞬間執行命令，將 `10ns` (範例) 加到延遲中，然後回傳。
3.  **發起者**: 重新獲得控制權。此時 `delay` 為 `10ns`。呼叫 `wait(10ns)`。
4.  **模擬**: 時間推進到 T=10ns。

## 5. 關鍵重點
-   **簡單性**: 程式碼是線通性的且易於閱讀。
-   **性能**: 由於目標不呼叫 `wait()`，因此最大限度地減少了上下文切換。
-   **責任**: **發起者** 負責時間同步 (`wait`)。
