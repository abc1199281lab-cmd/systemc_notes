# TLM Simple Initiator Socket 分析

> **檔案**: `ref/systemc/src/tlm_utils/simple_initiator_socket.h`

## 1. 概述
`simple_initiator_socket` 是 `tlm_utils` 中的一個工具類別，簡化了發起者 (initiators) 的實作。

## 2. 解決的問題
標準的 TLM sockets 要求擁有者模組繼承自 `tlm_bw_transport_if` 並實作 `nb_transport_bw` 和 `invalidate_direct_mem_ptr`。如果模組有多個 initiator sockets，這會導致多重繼承和名稱衝突。

## 3. 機制
- **回調註冊 (Callback Registration)**: `simple_initiator_socket` 改用註冊成員函式作為反向傳輸回調的方法，而非透過繼承。
  ```cpp
  socket.register_nb_transport_bw(this, &MyModule::my_nb_transport_bw);
  ```
- **內部配接器 (Internal Adapter)**: socket 包含一個內部的 `process` 類別，該類別繼承自 BW 介面，並將呼叫委發給註冊的回調。

## 4. 變體 (Variants)
- **`simple_initiator_socket<MODULE>`**: 標準版本。
- **`simple_initiator_socket_tagged<MODULE>`**: 用於一個模組有多個 sockets 綁定到 *同一個* 回調的情況。回調會收到一個整數 `id` 來識別是哪個 socket 觸發了呼叫。
  ```cpp
  void my_nb_transport_bw(int id, ...);
  ```

## 5. 關鍵重點
1.  **程式碼更簡潔**: 消除模組繼承傳輸介面的需求。
2.  **多傳輸端支援**: 標記 (tagged) 版本讓使用單一回調函式處理多個連接埠變得容易。
