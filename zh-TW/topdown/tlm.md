# 交易層建模 (TLM)

## 生活類比：快遞物流系統

想像整個快遞物流系統：

- **RTL 級建模** = 追蹤每一個包裹在每一秒鐘的確切位置——非常精確但極其費力
- **TLM 建模** = 只追蹤「包裹從倉庫A發出，3天後到達倉庫B」——快速且足夠精確
- **Generic Payload** = 標準化的包裹——不管裡面裝什麼，外面都有統一的運單格式
- **Socket** = 倉庫的收發窗口——「發貨窗口」(initiator) 和「收貨窗口」(target)
- **Transaction** = 一次包裹的寄送——從發送到接收的完整過程
- **Temporal Decoupling** = 快遞員一次帶走多個包裹——不用每個包裹都跑一趟

在晶片設計的早期階段，你不需要知道每根導線的時序，
只需要知道「CPU 讀了記憶體位址 0x1000，得到了資料 0xDEADBEEF，花了 100ns」。

---

## 什麼是 TLM？為什麼需要它？

### RTL 太慢的問題

```mermaid
graph TD
    subgraph "RTL 級模擬"
        R1["每個時鐘週期都模擬"]
        R2["每根導線都追蹤"]
        R3["非常精確"]
        R4["非常慢<br/>一個 SoC 可能要跑數小時"]
    end

    subgraph "TLM 模擬"
        T1["只模擬交易（讀/寫）"]
        T2["忽略信號級細節"]
        T3["精度足夠"]
        T4["非常快<br/>快 100~1000 倍"]
    end

    R4 -->|"不夠快！<br/>軟體團隊等不及"| T4

    style R4 fill:#ffcdd2
    style T4 fill:#c8e6c9
```

### TLM 的應用場景

```mermaid
flowchart TD
    ARCH["架構探索<br/>比較不同的設計方案"] --> TLM_USE["使用 TLM"]
    SW["軟體開發<br/>在硬體完成前<br/>就開始寫驅動程式"] --> TLM_USE
    PERF["效能分析<br/>頻寬、延遲、競爭"] --> TLM_USE
    VERIFY["功能驗證<br/>整體行為是否正確"] --> TLM_USE

    style TLM_USE fill:#c8e6c9
```

---

## TLM 1.0 vs TLM 2.0

```mermaid
graph LR
    subgraph "TLM 1.0"
        T1_PUT["put / get 介面"]
        T1_FIFO["tlm_fifo"]
        T1_ANALYSIS["tlm_analysis_port"]
        T1_DESC["較簡單<br/>類似 FIFO 的交易傳遞"]
    end

    subgraph "TLM 2.0"
        T2_GP["Generic Payload"]
        T2_SOCKET["Socket (initiator/target)"]
        T2_PHASE["Phase (BEGIN_REQ/END_REQ/...)"]
        T2_BT["Blocking Transport"]
        T2_NBT["Non-blocking Transport"]
        T2_DESC["更完整<br/>支援時序精度控制"]
    end

    T1_DESC -->|"演進"| T2_DESC

    style T1_DESC fill:#e3f2fd
    style T2_DESC fill:#c8e6c9
```

| 特性 | TLM 1.0 | TLM 2.0 |
|------|---------|---------|
| 主要介面 | put/get/peek | b_transport / nb_transport_fw/bw |
| 資料格式 | 任意型別 | Generic Payload (標準化) |
| 時序模型 | 無 | 有 (AT/LT) |
| 用途 | 簡單的資料傳遞 | 完整的匯流排建模 |
| 標準化 | 基本 | IEEE 1666-2011 |

---

## Generic Payload（通用載荷）

Generic Payload 是 TLM 2.0 定義的標準化交易格式，
就像國際物流的「國際運單」——不管寄什麼、從哪寄到哪，格式都一樣。

```mermaid
classDiagram
    class tlm_generic_payload {
        -m_command : tlm_command
        -m_address : uint64
        -m_data : unsigned char*
        -m_length : unsigned int
        -m_response_status : tlm_response_status
        -m_byte_enable : unsigned char*
        -m_byte_enable_length : unsigned int
        -m_streaming_width : unsigned int
        -m_extensions : vector
        +set_command(cmd)
        +set_address(addr)
        +set_data_ptr(data)
        +set_data_length(len)
        +set_response_status(status)
        +get_command() tlm_command
        +get_address() uint64
        +get_data_ptr() unsigned char*
    }

    class tlm_command {
        <<enumeration>>
        TLM_READ_COMMAND
        TLM_WRITE_COMMAND
        TLM_IGNORE_COMMAND
    }

    class tlm_response_status {
        <<enumeration>>
        TLM_OK_RESPONSE
        TLM_INCOMPLETE_RESPONSE
        TLM_ADDRESS_ERROR_RESPONSE
        TLM_GENERIC_ERROR_RESPONSE
    }

    tlm_generic_payload --> tlm_command
    tlm_generic_payload --> tlm_response_status
```

### 常用欄位解釋

```mermaid
flowchart TD
    GP["Generic Payload"]

    CMD["Command<br/>READ 或 WRITE"]
    ADDR["Address<br/>目標位址 (例: 0x1000)"]
    DATA["Data Pointer<br/>指向資料的指標"]
    LEN["Data Length<br/>資料長度 (位元組)"]
    RESP["Response Status<br/>交易是否成功"]
    BE["Byte Enable<br/>哪些位元組有效"]
    SW["Streaming Width<br/>串流模式寬度"]

    GP --> CMD
    GP --> ADDR
    GP --> DATA
    GP --> LEN
    GP --> RESP
    GP --> BE
    GP --> SW

    style CMD fill:#e3f2fd
    style ADDR fill:#e3f2fd
    style DATA fill:#e3f2fd
    style LEN fill:#e3f2fd
    style RESP fill:#c8e6c9
    style BE fill:#fff3e0
    style SW fill:#fff3e0
```

### 使用範例

```cpp
// 建立一個讀取交易
tlm::tlm_generic_payload trans;
unsigned char data[4];

trans.set_command(tlm::TLM_READ_COMMAND);
trans.set_address(0x1000);
trans.set_data_ptr(data);
trans.set_data_length(4);
trans.set_response_status(tlm::TLM_INCOMPLETE_RESPONSE);

// 發送交易
sc_time delay = SC_ZERO_TIME;
socket->b_transport(trans, delay);

// 檢查結果
if (trans.get_response_status() == tlm::TLM_OK_RESPONSE) {
    // data[] 現在包含從 0x1000 讀到的 4 個位元組
}
```

---

## Socket：發起者與目標

### 基本概念

```mermaid
flowchart LR
    subgraph "Initiator Module<br/>(CPU)"
        I_PROC["Process"]
        I_SOCK["initiator_socket"]
    end

    subgraph "Target Module<br/>(Memory)"
        T_SOCK["target_socket"]
        T_IMPL["b_transport 實作"]
    end

    I_PROC -->|"呼叫 b_transport"| I_SOCK
    I_SOCK -->|"forward path<br/>(請求)"| T_SOCK
    T_SOCK -->|"呼叫"| T_IMPL
    T_IMPL -->|"backward path<br/>(回應)"| I_SOCK

    style I_SOCK fill:#c8e6c9
    style T_SOCK fill:#fff3e0
```

### Socket 的類別結構

```mermaid
classDiagram
    class tlm_fw_transport_if~TYPES~ {
        <<abstract>>
        +b_transport(trans, delay)*
        +nb_transport_fw(trans, phase, delay)*
        +get_direct_mem_ptr(trans, dmi_data)*
        +transport_dbg(trans)*
    }

    class tlm_bw_transport_if~TYPES~ {
        <<abstract>>
        +nb_transport_bw(trans, phase, delay)*
        +invalidate_direct_mem_ptr(start, end)*
    }

    class tlm_initiator_socket~BUSWIDTH~ {
        +bind(target_socket)
        +operator->() fw_if
    }

    class tlm_target_socket~BUSWIDTH~ {
        +bind(initiator_socket)
    }

    tlm_initiator_socket --> tlm_fw_transport_if : uses
    tlm_initiator_socket --> tlm_bw_transport_if : provides
    tlm_target_socket --> tlm_fw_transport_if : provides
    tlm_target_socket --> tlm_bw_transport_if : uses
```

### 綁定方式

```cpp
// 直接綁定
initiator.socket.bind(target.socket);

// 或用運算子
initiator.socket(target.socket);
```

```mermaid
flowchart LR
    subgraph "系統架構範例"
        CPU["CPU<br/>initiator_socket"]
        BUS["Bus<br/>target + initiator"]
        MEM["Memory<br/>target_socket"]
        UART["UART<br/>target_socket"]
    end

    CPU -->|bind| BUS
    BUS -->|bind| MEM
    BUS -->|bind| UART

    style CPU fill:#c8e6c9
    style BUS fill:#e3f2fd
    style MEM fill:#fff3e0
    style UART fill:#fff3e0
```

---

## 兩種傳輸模式

### Blocking Transport（阻塞傳輸）

```cpp
void b_transport(tlm::tlm_generic_payload& trans, sc_time& delay);
```

整個交易在一個函式呼叫中完成，就像打電話——
撥號、等待接通、對話、掛斷，全部在一通電話中完成。

```mermaid
sequenceDiagram
    participant I as Initiator
    participant T as Target

    I->>T: b_transport(trans, delay)
    Note over T: 處理交易<br/>(讀/寫記憶體)
    Note over T: 設定 response_status
    Note over T: 累加 delay
    T-->>I: 返回
    Note over I: 檢查結果<br/>wait(delay)
```

### Non-blocking Transport（非阻塞傳輸）

```cpp
tlm_sync_enum nb_transport_fw(tlm_generic_payload& trans,
                               tlm_phase& phase,
                               sc_time& delay);
```

交易分多個階段完成，就像寄快遞——
下單、攬件、運送、派件、簽收，每個階段獨立。

```mermaid
sequenceDiagram
    participant I as Initiator
    participant T as Target

    I->>T: nb_transport_fw(trans, BEGIN_REQ, delay)
    Note over T: 收到請求

    T->>I: nb_transport_bw(trans, END_REQ, delay)
    Note over I: 請求被接受

    Note over T: 處理中...

    T->>I: nb_transport_bw(trans, BEGIN_RESP, delay)
    Note over I: 收到回應

    I->>T: nb_transport_fw(trans, END_RESP, delay)
    Note over T: 回應已確認
```

---

## Loosely-Timed vs Approximately-Timed

### Loosely-Timed (LT) — 粗略時序

```mermaid
flowchart TD
    LT["Loosely-Timed (LT)"]
    LT1["使用 b_transport"]
    LT2["時序不太精確"]
    LT3["模擬速度最快"]
    LT4["適合軟體開發和早期探索"]
    LT5["支援 temporal decoupling"]

    LT --> LT1
    LT --> LT2
    LT --> LT3
    LT --> LT4
    LT --> LT5

    style LT fill:#c8e6c9
```

### Approximately-Timed (AT) — 近似時序

```mermaid
flowchart TD
    AT["Approximately-Timed (AT)"]
    AT1["使用 nb_transport"]
    AT2["時序較精確"]
    AT3["模擬速度較慢"]
    AT4["適合效能分析"]
    AT5["可模擬管線和並行"]

    AT --> AT1
    AT --> AT2
    AT --> AT3
    AT --> AT4
    AT --> AT5

    style AT fill:#fff3e0
```

### 精度與速度的取捨

```mermaid
graph LR
    FAST["模擬速度"] -->|"高"| LT["Loosely-Timed<br/>(b_transport)"]
    FAST -->|"中"| AT["Approximately-Timed<br/>(nb_transport)"]
    FAST -->|"低"| RTL["RTL<br/>(signal-level)"]

    ACCURATE["時序精度"] -->|"低"| LT
    ACCURATE -->|"中"| AT
    ACCURATE -->|"高"| RTL

    style LT fill:#c8e6c9
    style AT fill:#fff3e0
    style RTL fill:#ffcdd2
```

---

## Phase（交易階段）

TLM 2.0 定義了四個基本階段：

```mermaid
stateDiagram-v2
    [*] --> BEGIN_REQ: Initiator 發起請求
    BEGIN_REQ --> END_REQ: Target 接受請求
    END_REQ --> BEGIN_RESP: Target 開始回應
    BEGIN_RESP --> END_RESP: Initiator 接受回應
    END_RESP --> [*]: 交易完成

    state BEGIN_REQ {
        [*] --> 請求開始
        請求開始 --> 地址和命令傳送中
    }

    state END_REQ {
        [*] --> 請求結束
        請求結束 --> Target可以處理了
    }

    state BEGIN_RESP {
        [*] --> 回應開始
        回應開始 --> 資料傳送中
    }

    state END_RESP {
        [*] --> 回應結束
        回應結束 --> Initiator收到資料了
    }
```

### Phase 對應到匯流排行為

```mermaid
sequenceDiagram
    participant CPU as CPU (Initiator)
    participant BUS as Bus
    participant MEM as Memory (Target)

    Note over CPU, MEM: BEGIN_REQ
    CPU->>BUS: 位址 + 命令放上匯流排

    Note over CPU, MEM: END_REQ
    BUS->>MEM: 仲裁完成，請求送達

    Note over CPU, MEM: BEGIN_RESP
    MEM->>BUS: 資料放上匯流排

    Note over CPU, MEM: END_RESP
    BUS->>CPU: 資料送達 CPU
```

---

## Temporal Decoupling 與 Quantum

### 問題：同步太頻繁

在正常的模擬中，每次交易都要和引擎同步（呼叫 `wait()`），
這會嚴重拖慢模擬速度。

### 解決：讓 process 跑得比模擬時間快

```mermaid
flowchart TD
    subgraph "沒有 Temporal Decoupling"
        N1["交易 1 → wait(10ns)"]
        N2["交易 2 → wait(10ns)"]
        N3["交易 3 → wait(10ns)"]
        N4["每次都要和引擎同步<br/>很慢！"]
        N1 --> N2 --> N3 --> N4
    end

    subgraph "有 Temporal Decoupling"
        T1["交易 1 → local_time += 10ns"]
        T2["交易 2 → local_time += 10ns"]
        T3["交易 3 → local_time += 10ns"]
        T4["local_time > quantum?<br/>才和引擎同步一次"]
        T1 --> T2 --> T3 --> T4
    end

    style N4 fill:#ffcdd2
    style T4 fill:#c8e6c9
```

### Quantum Keeper

```mermaid
classDiagram
    class tlm_quantumkeeper {
        -m_local_time : sc_time
        -m_next_sync_point : sc_time
        +set(local_time)
        +inc(delta)
        +get_local_time() sc_time
        +get_current_time() sc_time
        +need_sync() bool
        +sync()
        +reset()
    }

    class tlm_global_quantum {
        -m_global_quantum : sc_time
        +set(quantum)
        +get() sc_time
        +compute_local_quantum() sc_time
        +instance() tlm_global_quantum
    }

    tlm_quantumkeeper --> tlm_global_quantum : reads
```

### 使用範例

```cpp
// 設定 global quantum (例如 1 微秒)
tlm::tlm_global_quantum::instance().set(sc_time(1, SC_US));

// 在 initiator 中
tlm_utils::tlm_quantumkeeper qk;
qk.reset();

void run() {
    while (true) {
        // 執行交易
        sc_time delay = SC_ZERO_TIME;
        socket->b_transport(trans, delay);

        // 累加本地時間
        qk.inc(delay);

        // 檢查是否需要同步
        if (qk.need_sync()) {
            qk.sync();  // wait(local_time)
        }
    }
}
```

```mermaid
sequenceDiagram
    participant I as Initiator
    participant QK as QuantumKeeper
    participant SIM as 模擬引擎

    Note over SIM: 模擬時間 = 0ns
    Note over QK: quantum = 100ns

    I->>QK: inc(10ns) [交易1]
    Note over QK: local_time = 10ns

    I->>QK: inc(20ns) [交易2]
    Note over QK: local_time = 30ns

    I->>QK: inc(40ns) [交易3]
    Note over QK: local_time = 70ns

    I->>QK: inc(50ns) [交易4]
    Note over QK: local_time = 120ns > quantum!

    QK->>QK: need_sync() = true
    QK->>SIM: sync() → wait(120ns)
    Note over SIM: 模擬時間 = 120ns
    Note over QK: local_time = 0ns
```

---

## DMI（Direct Memory Interface）

DMI 允許 initiator 直接存取 target 的記憶體，
跳過 transport 介面——就像快遞員直接把鑰匙給你，以後你自己去倉庫拿。

```mermaid
flowchart TD
    subgraph "普通 transport"
        P1["每次讀寫都要<br/>呼叫 b_transport"]
        P2["經過 socket → target"]
        P3["較慢但有完整追蹤"]
    end

    subgraph "DMI"
        D1["第一次請求 DMI 指標"]
        D2["之後直接用指標<br/>讀寫記憶體"]
        D3["極快！但沒有追蹤"]
    end

    P1 --> P2 --> P3
    D1 --> D2 --> D3

    style P3 fill:#fff3e0
    style D3 fill:#c8e6c9
```

---

## TLM 2.0 便利 Socket

`tlm_utils` 提供了簡化版的 socket，減少樣板程式碼：

```mermaid
classDiagram
    class simple_initiator_socket {
        +register_nb_transport_bw(callback)
        +register_invalidate_direct_mem_ptr(callback)
    }

    class simple_target_socket {
        +register_b_transport(callback)
        +register_nb_transport_fw(callback)
        +register_get_direct_mem_ptr(callback)
        +register_transport_dbg(callback)
    }

    class multi_passthrough_initiator_socket {
        +register_nb_transport_bw(callback)
        +bind(target)
    }

    class multi_passthrough_target_socket {
        +register_b_transport(callback)
        +register_nb_transport_fw(callback)
    }

    note for simple_target_socket "最常用！<br/>只需要註冊 b_transport<br/>回呼函式即可"
```

---

## 完整的 TLM 2.0 系統範例

```mermaid
flowchart TD
    subgraph "SoC 平台模型"
        CPU["CPU<br/>simple_initiator_socket"]
        DMA["DMA<br/>initiator + target"]
        BUS["Interconnect<br/>multi_passthrough<br/>target + initiator"]
        RAM["RAM<br/>simple_target_socket<br/>+ DMI 支援"]
        UART["UART<br/>simple_target_socket"]
        GPIO["GPIO<br/>simple_target_socket"]
    end

    CPU -->|"0x0000-0xFFFF"| BUS
    DMA -->|"0x0000-0xFFFF"| BUS
    BUS -->|"0x0000-0x7FFF"| RAM
    BUS -->|"0x8000-0x800F"| UART
    BUS -->|"0x8010-0x801F"| GPIO
    CPU -->|"設定 DMA"| DMA

    style CPU fill:#c8e6c9
    style DMA fill:#e3f2fd
    style BUS fill:#fff3e0
    style RAM fill:#fce4ec
    style UART fill:#fce4ec
    style GPIO fill:#fce4ec
```

---

## 相關模組

| 概念 | 文件 | 關係 |
|------|------|------|
| 通訊機制 | [communication.md](communication.md) | TLM 是更高層次的通訊抽象 |
| 模組階層 | [hierarchy.md](hierarchy.md) | TLM 模組仍然是 sc_module |
| 事件機制 | [events.md](events.md) | Temporal decoupling 減少事件同步 |
| 排程機制 | [scheduling.md](scheduling.md) | Quantum 影響排程頻率 |

### 對應的底層程式碼文件

| 原始碼概念 | 程式碼文件 |
|-----------|-----------|
| tlm_generic_payload | [doc_v2/code/tlm_core/tlm_2/tlm_generic_payload.md](../code/tlm_core/tlm_2/tlm_generic_payload.md) |
| tlm_phase | [doc_v2/code/tlm_core/tlm_2/tlm_phase.md](../code/tlm_core/tlm_2/tlm_phase.md) |
| tlm_fw_bw_ifs | [doc_v2/code/tlm_core/tlm_2/tlm_fw_bw_ifs.md](../code/tlm_core/tlm_2/tlm_fw_bw_ifs.md) |
| tlm_initiator_socket | [doc_v2/code/tlm_core/tlm_2/tlm_initiator_socket.md](../code/tlm_core/tlm_2/tlm_initiator_socket.md) |
| tlm_target_socket | [doc_v2/code/tlm_core/tlm_2/tlm_target_socket.md](../code/tlm_core/tlm_2/tlm_target_socket.md) |
| tlm_global_quantum | [doc_v2/code/tlm_core/tlm_2/tlm_global_quantum.md](../code/tlm_core/tlm_2/tlm_global_quantum.md) |
| tlm_dmi | [doc_v2/code/tlm_core/tlm_2/tlm_dmi.md](../code/tlm_core/tlm_2/tlm_dmi.md) |
| tlm_quantumkeeper | [doc_v2/code/tlm_utils/tlm_quantumkeeper.md](../code/tlm_utils/tlm_quantumkeeper.md) |
| simple_initiator_socket | [doc_v2/code/tlm_utils/simple_initiator_socket.md](../code/tlm_utils/simple_initiator_socket.md) |
| simple_target_socket | [doc_v2/code/tlm_utils/simple_target_socket.md](../code/tlm_utils/simple_target_socket.md) |
| tlm_analysis | [doc_v2/code/tlm_core/tlm_1/tlm_analysis.md](../code/tlm_core/tlm_1/tlm_analysis.md) |
| tlm_req_rsp | [doc_v2/code/tlm_core/tlm_1/tlm_req_rsp.md](../code/tlm_core/tlm_1/tlm_req_rsp.md) |
| peq_with_cb_and_phase | [doc_v2/code/tlm_utils/peq_with_cb_and_phase.md](../code/tlm_utils/peq_with_cb_and_phase.md) |

---

## 學習小提示

1. **先學 b_transport，再學 nb_transport**——大多數情況下 LT 模式就夠了
2. **Generic Payload 是 TLM 2.0 的核心**——理解它的每個欄位
3. **Socket = Port + Export 的結合體**——它同時提供了 forward 和 backward 路徑
4. **Temporal Decoupling 是速度的關鍵**——不用它，TLM 的速度優勢會大打折扣
5. **DMI 讓記憶體存取飛快**——對記憶體密集的模擬非常重要
6. **TLM 和 RTL 可以混合使用**——用 TLM 建模大部分系統，只對關注的部分用 RTL
7. **simple_target_socket 是你的好朋友**——它省去了很多樣板程式碼
8. **Extension 機制讓 Generic Payload 可擴展**——需要自訂欄位時不用修改 payload 本身
