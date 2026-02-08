# TLM Target Socket 分析

> **檔案**: `ref/systemc/src/tlm_core/tlm_2/tlm_sockets/tlm_target_socket.h`

## 1. 概述
`tlm_target_socket` 是發起者通訊端 (initiator socket) 的對應組件，設計由目標模組 (targets/slaves/peripherals) 暴露。

## 2. 結構
它包裝了：
- **`sc_export`**: 用於 **正向路徑 (Forward Path)** (接收來自發起者的呼叫)。
- **`sc_port`**: 用於 **反向路徑 (Backward Path)** (將呼叫傳回給發起者)。

## 3. 綁定 (Binding)
標準的目標通訊端綁定可以接受一個 `initiator_socket`：
```cpp
target_socket.bind( initiator_socket ); // 較少見，通常由 initiator 綁定到 target
```
或者允許階層式綁定 (父模組的目標通訊端與子模組的目標通訊端綁定)。

## 4. 關鍵重點
1.  **對稱性 (Symmetry)**: 其結構鏡像了發起者通訊端，維護了 SystemC 綁定機制運作所需的 `port/export` 二元性。
2.  **版本化**: TLM 2.0 sockets 使用大量的模板 (`BUSWIDTH`, `TYPES`, `N`, `POL`) 以支援各種匯流排寬度和協定型別，儘管預設值涵蓋了 90% 的使用案例。
