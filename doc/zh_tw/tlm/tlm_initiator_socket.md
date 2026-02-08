# TLM Initiator Socket 分析

> **檔案**: `ref/systemc/src/tlm_core/tlm_2/tlm_sockets/tlm_initiator_socket.h`

## 1. 概述
`tlm_initiator_socket` 是一個便利類別，它將 TLM 2.0 雙向通訊所需的正向 (forward) 和反向 (backward) 路徑打包在一起。

## 2. 結構
在 TLM 2.0 之前，連接發起者 (initiator) 和目標 (target) 需要兩個獨立的綁定：一個用於請求 (發起者 -> 目標)，另一個用於回應 (目標 -> 發起者)。
Socket 包裝了：
- **`sc_port`**: 用於 **正向路徑 (Forward Path)** (發起者在目標上呼叫 `nb_transport_fw`, `b_transport` 等)。
- **`sc_export`**: 用於 **反向路徑 (Backward Path)** (目標在發起者上呼叫 `nb_transport_bw`)。

## 3. 綁定 (Binding)
Socket 覆寫了 `bind()` 方法 (以及 `operator()`) 以執行 "雙重綁定"：
```cpp
initiator_socket.bind( target_socket );
```
這行代碼在內部執行：
1.  將發起者的港口 (port) 綁定到目標的匯出 (export)。
2.  將目標的港口 (port) 綁定到發起者的匯出 (export)。

## 4. 關鍵重點
1.  **簡單性**: 極大地減少了連接模組所需的樣板程式碼 (boilerplate code)。
2.  **安全性**: 確保路徑雙向連接，防止出現目標試圖向斷開連接的發起者發送回應的不完全連接情況。
