# SystemC Interface 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_interface.cpp`

## 1. 概述
`sc_interface` 是 SystemC 中所有介面類別的抽象基底類別。介面定義了通道 (channel) 必須實作以及連接埠 (port) 可以呼叫的一組操作 (methods)。

## 2. 核心方法

### 2.1 `register_port`
- **簽章 (Signature)**: `virtual void register_port(sc_port_base& port, const char* if_typename)`
- **目的**: 當連接埠綁定到此介面時被呼叫。
- **預設**: 什麼都不做。通道可以覆寫此函式以追蹤哪些連接埠連線到它們 (例如，用於仲裁或連通性檢查)。

### 2.2 `default_event`
- **簽章 (Signature)**: `virtual const sc_event& default_event() const`
- **目的**: 返回與此介面關聯的預設事件。
- **使用案例**: 由 `sensitive << port;` 使用。如果一個行程對連接埠敏感，它實際上是對介面的 `default_event()` 敏感。
- **預設**: 警告 `SC_ID_NO_DEFAULT_EVENT_` 並返回 `sc_event::none`。

---

## 3. 關鍵重點
1.  **純虛擬解構子 (Pure Virtual Destructor)**: 它確保衍生通道物件的正確清理。
2.  **合約 (Contract)**: 它嚴格定義了通訊通道的 "API" 部分。
