# communication -- SystemC 通訊子系統

SystemC 的 `communication` 子系統定義了模組之間溝通的所有基礎設施。它實現了 **介面 (Interface)**、**通道 (Channel)**、**埠 (Port)** 和 **匯出 (Export)** 四大核心概念，讓硬體模組能夠互相傳遞資料和事件。

## 概述

想像一座城市的郵政系統：
- **介面 (Interface)** 就像「寄信」和「收信」的規則 -- 定義了能做什麼操作
- **通道 (Channel)** 就像實際的郵筒和信箱 -- 實現了資料的儲存和傳遞
- **埠 (Port)** 就像每棟建築物門口的信箱口 -- 模組透過它來存取外部的通道
- **匯出 (Export)** 就像「服務窗口」 -- 讓模組把內部的介面暴露給外部使用

這套架構遵循「介面與實現分離」的設計原則：模組只依賴介面定義，不依賴具體的通道實現，從而實現鬆耦合 (loose coupling)。

## 架構關係圖

```mermaid
graph TB
    subgraph "核心抽象層"
        IF[sc_interface<br/>所有介面的基底類別]
        PC[sc_prim_channel<br/>原始通道基底類別]
        PB[sc_port_base<br/>埠的基底類別]
        EB[sc_export_base<br/>匯出的基底類別]
    end

    subgraph "訊號介面層"
        SIF_IN[sc_signal_in_if&lt;T&gt;<br/>輸入介面]
        SIF_WRITE[sc_signal_write_if&lt;T&gt;<br/>寫入介面]
        SIF_INOUT[sc_signal_inout_if&lt;T&gt;<br/>輸入輸出介面]
    end

    subgraph "訊號通道層"
        SIG[sc_signal&lt;T&gt;<br/>訊號通道]
        BUF[sc_buffer&lt;T&gt;<br/>緩衝通道]
        CLK[sc_clock<br/>時脈通道]
    end

    subgraph "訊號埠層"
        SP_IN[sc_in&lt;T&gt;<br/>輸入埠]
        SP_INOUT[sc_inout&lt;T&gt;<br/>輸入輸出埠]
        SP_OUT[sc_out&lt;T&gt;<br/>輸出埠]
    end

    subgraph "輔助設施"
        EF[sc_event_finder<br/>事件尋找器]
        EQ[sc_event_queue<br/>事件佇列]
        WP[sc_writer_policy<br/>寫入策略]
        CID[sc_communication_ids<br/>錯誤訊息 ID]
    end

    IF --> PC
    IF --> SIF_IN
    IF --> SIF_WRITE
    SIF_IN --> SIF_INOUT
    SIF_WRITE --> SIF_INOUT
    PC --> SIG
    SIG --> BUF
    SIG --> CLK
    SIF_INOUT -.->|實現| SIG
    PB --> SP_IN
    PB --> SP_INOUT
    SP_INOUT --> SP_OUT
    WP -.->|策略| SIG
    EF -.->|輔助| PB
```

## 資料流示意圖

```mermaid
sequenceDiagram
    participant Module_A as 模組 A
    participant Port_Out as sc_out (輸出埠)
    participant Signal as sc_signal (訊號通道)
    participant Port_In as sc_in (輸入埠)
    participant Module_B as 模組 B

    Module_A->>Port_Out: write(value)
    Port_Out->>Signal: write(value) via interface
    Note over Signal: 值暫存到 m_new_val<br/>呼叫 request_update()
    Note over Signal: === update phase ===
    Signal->>Signal: update(): m_cur_val = m_new_val
    Signal->>Signal: notify value_changed_event
    Signal->>Port_In: 事件觸發
    Port_In->>Module_B: 喚醒敏感的 process
    Module_B->>Port_In: read()
    Port_In->>Signal: read() via interface
    Signal-->>Module_B: 回傳 m_cur_val
```

## 檔案列表

| 檔案 | 說明 |
|------|------|
| [sc_interface.md](sc_interface.md) | 所有介面類別的抽象基底類別 |
| [sc_port.md](sc_port.md) | 埠的基底類別，模組存取外部通道的入口 |
| [sc_export.md](sc_export.md) | 匯出的基底類別，讓模組暴露內部介面 |
| [sc_prim_channel.md](sc_prim_channel.md) | 原始通道的抽象基底類別 |
| [sc_signal.md](sc_signal.md) | 泛型訊號通道 `sc_signal<T>` |
| [sc_signal_ifs.md](sc_signal_ifs.md) | 訊號相關的介面定義 |
| [sc_signal_ports.md](sc_signal_ports.md) | 訊號專用的埠類別 (`sc_in`, `sc_inout`, `sc_out`) |
| [sc_writer_policy.md](sc_writer_policy.md) | 訊號寫入策略，控制多重寫入者行為 |
| [sc_buffer.md](sc_buffer.md) | 緩衝通道，每次寫入都觸發事件 |
| [sc_clock.md](sc_clock.md) | 時脈通道，產生週期性的布林訊號 |
| [sc_clock_ports.md](sc_clock_ports.md) | 時脈專用埠的型別別名 |
| [sc_event_finder.md](sc_event_finder.md) | 事件尋找器，在綁定完成前延遲解析事件 |
| [sc_event_queue.md](sc_event_queue.md) | 事件佇列，支援多個待處理通知 |
| [sc_communication_ids.md](sc_communication_ids.md) | 通訊子系統的錯誤/警告訊息 ID |

## 依賴關係圖

```mermaid
graph LR
    sc_interface --> sc_port
    sc_interface --> sc_export
    sc_interface --> sc_signal_ifs
    sc_interface --> sc_event_queue

    sc_signal_ifs --> sc_signal
    sc_signal_ifs --> sc_signal_ports

    sc_port --> sc_signal_ports
    sc_port --> sc_event_finder

    sc_prim_channel --> sc_signal

    sc_signal --> sc_buffer
    sc_signal --> sc_clock

    sc_writer_policy --> sc_signal
    sc_writer_policy --> sc_signal_ifs

    sc_signal_ports --> sc_clock_ports

    sc_event_finder --> sc_signal_ports

    sc_communication_ids -.->|錯誤訊息| sc_port
    sc_communication_ids -.->|錯誤訊息| sc_export
    sc_communication_ids -.->|錯誤訊息| sc_signal
    sc_communication_ids -.->|錯誤訊息| sc_prim_channel
```

## 設計理念

### 為什麼要分離介面、通道和埠？

在真正的硬體設計中，模組之間的連線（wire）和通訊協定（protocol）是分開的概念。SystemC 沿用了這個思想：

1. **介面 (Interface)** 定義「能做什麼」（例如：讀取、寫入）
2. **通道 (Channel)** 定義「怎麼做」（例如：用兩個暫存器實現 delta cycle 更新）
3. **埠 (Port)** 定義「從哪裡存取」（連接模組與通道的橋樑）
4. **匯出 (Export)** 定義「提供什麼服務」（讓子模組的介面對外可見）

這種分離讓同一個介面可以有不同的通道實現，也讓模組可以在不知道具體通道類型的情況下進行通訊。

## 相關目錄

- `sysc/kernel/` - 核心模擬引擎（事件、排程器）
- `sysc/datatypes/` - 資料型別（`sc_logic`, `sc_bv` 等）
- `sysc/tracing/` - 波形追蹤
