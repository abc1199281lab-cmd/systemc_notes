# TLM Simple Target Socket 分析

> **檔案**: `ref/systemc/src/tlm_utils/simple_target_socket.h`

## 1. 概述
`simple_target_socket` 是 `tlm_utils` 中功能強大的工具，具備簡化目標實作的功能，並提供自動傳輸配接 (automatic transport adaptation)。

## 2. 動態配接 (Dynamic Adaptation)
此 socket 可以在阻塞 (Blocking) 與非阻塞 (Non-Blocking) 呼叫之間進行轉換。
- **阻塞目標，非阻塞發起者**: 如果目標僅註冊了 `b_transport` 但發起者呼叫了 `nb_transport_fw`，socket 的內部 `fw_process` 會：
    1.  攔截 `nb` 呼叫。
    2.  產生一個動態 `SC_THREAD` (`nb2b_thread`)。
    3.  在該執行緒中呼叫使用者的 `b_transport`。
    4.  當阻塞呼叫回傳時，處理回傳路徑 (發送 `BEGIN_RESP/END_RESP`)。
- **非阻塞目標，阻塞發起者**: 如果目標僅註冊了 `nb_transport` 但發起者呼叫了 `b_transport`，socket 會：
    1.  呼叫 `nb_transport_fw`。
    2.  在內部事件 (`m_peq`) 上等待，直到回應完成。

## 3. 回調註冊
與 simple initiator socket 類似，它允許註冊回調而非繼承自 `tlm_fw_transport_if`。
```cpp
socket.register_b_transport(this, &MyModule::my_b_transport);
```

## 4. 變體 (Variants)
- **`simple_target_socket<MODULE>`**: 標準版本。
- **`simple_target_socket_tagged<MODULE>`**: 在回調中加入整數 ID，適合使用單一函式處理多個 sockets (例如：多埠記憶體控制器)。

## 5. 關鍵重點
1.  **互操作性 (Interoperability)**: 讓簡單的阻塞目標 (易於撰寫) 能與複雜的非阻塞發起者 (高性能) 共同運作，而無需修改使用者程式碼。
2.  **轉換成本**: `nb` -> `b` 的轉換涉及產生執行緒，這會帶來效能損失。為了在 AT 模型中獲得最高速度，應使用原生的 `nb_transport`。
