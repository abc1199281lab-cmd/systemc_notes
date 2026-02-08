# TLM 多重透傳發起者通訊端 (Multi-Passthrough Initiator Socket) 分析

> **檔案**: `ref/systemc/src/tlm_utils/multi_passthrough_initiator_socket.h`

## 1. 概述
`multi_passthrough_initiator_socket` 允許單一發起者通訊端與多個目標進行綁定。這通常用於互連結構 (interconnects)、路由器 (routers) 或匯流排。

## 2. 機制
- **動態綁定**: 可與 $N$ 個目標綁定。綁定數量在 elaboration 時間確定。
- **索引**: Socket 維護一個綁定目標的向量。
    - `socket[i]` 回傳指向第 $i$ 個目標的介面。
    - `socket.size()` 回傳已綁定目標的數量。

## 3. 回調 (Callbacks)
由於反向路徑 (來自目標的回調) 需要識別是由 *哪一個* 目標回應，因此註冊的回調包含一個索引 (index)：
```cpp
// 使用者回調簽章
sync_enum_type nb_transport_bw(int id, transaction_type& trans, phase_type& phase, sc_time& t);

// 註冊
socket.register_nb_transport_bw(this, &MyModule::my_nb_transport_bw);
```
Socket 會自動為每個連線分配一個 ID (從 0 到 N-1)，並將此 ID 傳遞給回調函式。

## 4. 關鍵重點
1.  **互連結構**: 這是建立匯流排和路由器模型的基礎組件，在這些模型中，一個組件需要與多個對象通訊。
2.  **無轉換**: 與 `simple_target_socket` 不同，"透傳 (passthrough)" sockets **不** 執行阻塞/非阻塞轉換。它們嚴格用於路由呼叫。
