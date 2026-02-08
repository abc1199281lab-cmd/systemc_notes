# TLM 多重透傳目標通訊端 (Multi-Passthrough Target Socket) 分析

> **檔案**: `ref/systemc/src/tlm_utils/multi_passthrough_target_socket.h`

## 1. 概述
`multi_passthrough_target_socket` 允許將多個發起者綁定到單一目標通訊端。這通常用於共享資源 (如記憶體) 或互連結構中。

## 2. 機制
- **動態綁定**: 接受來自 $N$ 個發起者的連線。
- **索引**: `socket[i]` 提供對第 $i$ 個發起者反向路徑的存取。

## 3. 回調 (Callbacks)
來自發起者的正向路徑呼叫需要識別來源。註冊的回調包含一個索引：
```cpp
// 使用者回調簽章
void b_transport(int id, transaction_type& trans, sc_time& t);

// 註冊
socket.register_b_transport(this, &MyModule::my_b_transport);
```
當發起者 $K$ 呼叫 `b_transport` 時，socket 會以 `id = K` 啟動 `my_b_transport`。

## 4. 關鍵重點
1.  **仲裁/路由**: 對於需要區分多個流量來源的組件 (檢查 ID 以進行仲裁或地址映射) 來說非常重要。
2.  **簡易性**: 與發起者端的對應組件一樣，它不轉換交易型別，僅加入 ID 並轉發呼叫。
