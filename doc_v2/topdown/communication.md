# 通訊機制

## 生活類比：公司的部門間溝通

想像一家大公司裡不同部門之間的溝通方式：

- **Interface（介面）** = 公司的溝通規範——「報告要用什麼格式？會議怎麼預約？」
- **Channel（通道）** = 具體的溝通工具——Email、Slack、電話
- **Port（埠口）** = 每個部門的對外聯絡窗口——「找業務部請撥分機 101」
- **Signal** = 公佈欄——寫上新的公告，大家下次經過就能看到
- **FIFO** = 排隊的信箱——先寄來的信先處理
- **Mutex** = 會議室鑰匙——一次只有一個人可以用
- **Semaphore** = 停車場剩餘車位數——有限數量的共享資源

重點是：部門不需要知道對方用什麼電腦或軟體，
只需要遵守約定好的溝通規範（介面）。

---

## Interface-Channel-Port 模式

這是 SystemC 通訊機制的核心設計模式，
也是理解所有通訊元件的鑰匙。

```mermaid
flowchart TD
    subgraph "設計原則"
        IF["Interface<br/>定義「能做什麼」<br/>(純虛擬類別)"]
        CH["Channel<br/>實作「怎麼做」<br/>(繼承 Interface)"]
        PO["Port<br/>宣告「我需要什麼」<br/>(模板參數是 Interface)"]
    end

    PO -->|"透過 Interface 呼叫"| CH
    CH -->|"實作"| IF

    style IF fill:#e3f2fd
    style CH fill:#fff3e0
    style PO fill:#c8e6c9
```

### 為什麼要這樣設計？

```mermaid
flowchart LR
    subgraph "不好的設計 ✗"
        A1["Module A"] -->|"直接存取"| B1["Module B 的內部"]
    end

    subgraph "SystemC 的設計 ✓"
        A2["Module A<br/>的 Port"] -->|"透過 Interface"| C2["Channel"]
        C2 -->|"透過 Interface"| B2["Module B<br/>的 Port"]
    end

    style A1 fill:#ffcdd2
    style B1 fill:#ffcdd2
    style A2 fill:#c8e6c9
    style C2 fill:#e3f2fd
    style B2 fill:#c8e6c9
```

好處：
1. **解耦** — 模組不需要知道對方的存在
2. **可替換** — 換一種 Channel 實作，模組不用改
3. **可重用** — 同一個模組可以在不同系統中使用

---

## Signal 與 Request-Update 機制

`sc_signal` 是最基本的通訊 channel，對應硬體中的導線（wire）。

### Signal 的核心行為

```mermaid
sequenceDiagram
    participant P as Process
    participant S as sc_signal
    participant E as 模擬引擎

    Note over S: 當前值 = 0, 新值 = 0

    P->>S: write(1)
    Note over S: 當前值 = 0, 新值 = 1<br/>(新值暫存, 還沒生效！)
    S->>E: request_update()

    Note over E: Evaluate 階段結束
    Note over E: 進入 Update 階段

    E->>S: update()
    Note over S: 當前值 = 1, 新值 = 1<br/>(現在才生效！)
    S->>E: 值改變了！通知相關 process
```

### 為什麼 Signal 不能立即更新？

```mermaid
graph TD
    subgraph "如果立即更新 (錯誤!)"
        A1["Process A 寫入 sig = 1"]
        B1["Process B 讀取 sig = 1"]
        C1["Process C 讀取 sig = 1"]
        A1 --> B1 --> C1
        NOTE1["B 和 C 看到的值取決於<br/>執行順序！不確定！"]
    end

    subgraph "Request-Update (正確)"
        A2["Process A 寫入 sig (暫存)"]
        B2["Process B 讀取 sig = 舊值"]
        C2["Process C 讀取 sig = 舊值"]
        U2["Update: sig 更新為新值"]
        A2 --> U2
        B2 --> U2
        C2 --> U2
        NOTE2["B 和 C 都看到舊值<br/>不管執行順序，結果一致！"]
    end

    style NOTE1 fill:#ffcdd2
    style NOTE2 fill:#c8e6c9
```

### sc_signal 的類別結構

```mermaid
classDiagram
    class sc_interface {
        <<abstract>>
        +register_port()
        +default_event() sc_event
    }

    class sc_signal_in_if~T~ {
        <<abstract>>
        +read() T
        +value_changed_event() sc_event
    }

    class sc_signal_inout_if~T~ {
        <<abstract>>
        +write(T)
    }

    class sc_prim_channel {
        #request_update()
        #update()
        #async_request_update()
    }

    class sc_signal~T~ {
        -m_cur_val : T
        -m_new_val : T
        -m_change_event : sc_event
        +read() T
        +write(T)
        #update()
    }

    sc_interface <|-- sc_signal_in_if
    sc_signal_in_if <|-- sc_signal_inout_if
    sc_prim_channel <|-- sc_signal
    sc_signal_inout_if <|.. sc_signal
```

---

## FIFO — 先進先出佇列

`sc_fifo` 就像排隊買東西——先排的人先買到。

```mermaid
flowchart LR
    W["寫入端<br/>(Producer)"] -->|"write(data)"| FIFO["sc_fifo<br/>容量 = N<br/>[data3][data2][data1]"]
    FIFO -->|"read()"| R["讀取端<br/>(Consumer)"]

    FULL["滿了？ → write 會阻塞<br/>（等到有空位）"]
    EMPTY["空了？ → read 會阻塞<br/>（等到有資料）"]

    FIFO -.-> FULL
    FIFO -.-> EMPTY

    style W fill:#c8e6c9
    style R fill:#fff3e0
    style FIFO fill:#e3f2fd
```

### FIFO 的事件通知

```mermaid
sequenceDiagram
    participant Producer
    participant FIFO as sc_fifo (容量=2)
    participant Consumer

    Producer->>FIFO: write("A")
    Note over FIFO: ["A"]
    FIFO->>Consumer: data_written_event 通知

    Producer->>FIFO: write("B")
    Note over FIFO: ["A", "B"] (滿了！)

    Producer->>FIFO: write("C") — 阻塞！
    Note over Producer: 等待空位...

    Consumer->>FIFO: read() → "A"
    Note over FIFO: ["B"]
    FIFO->>Producer: data_read_event 通知
    Note over Producer: 有空位了！

    Producer->>FIFO: write("C") — 成功
    Note over FIFO: ["B", "C"]
```

### FIFO vs Signal 的差異

| 特性 | sc_signal | sc_fifo |
|------|-----------|---------|
| 資料保留 | 只保留最新值 | 保留所有值直到被讀取 |
| 阻塞 | 不阻塞 | 滿時寫入阻塞，空時讀取阻塞 |
| 適用場景 | 硬體信號線 | 資料流、生產者-消費者 |
| 多個讀取者 | 可以 | 不行（讀取會消耗資料） |
| Delta cycle | 需要（request-update） | 需要 |

---

## Mutex — 互斥鎖

`sc_mutex` 保證同一時間只有一個 process 可以存取共享資源。

```mermaid
sequenceDiagram
    participant A as Process A
    participant M as sc_mutex
    participant B as Process B

    A->>M: lock() — 成功！
    Note over M: 已鎖定 (擁有者: A)

    B->>M: lock() — 阻塞！
    Note over B: 等待...

    A->>M: unlock()
    Note over M: 已解鎖
    M->>B: 你可以用了！

    B->>M: lock() — 成功！
    Note over M: 已鎖定 (擁有者: B)
```

類比：圖書館的自習室只有一把鑰匙。
你用完了，把鑰匙還回去，下一個排隊的人才能進去。

---

## Semaphore — 號誌

`sc_semaphore` 允許最多 N 個 process 同時存取資源。

```mermaid
flowchart TD
    SEM["sc_semaphore<br/>初始值 = 3"]
    P1["Process 1<br/>wait() → 剩 2"]
    P2["Process 2<br/>wait() → 剩 1"]
    P3["Process 3<br/>wait() → 剩 0"]
    P4["Process 4<br/>wait() → 阻塞！"]

    SEM --> P1
    SEM --> P2
    SEM --> P3
    SEM -.->|"沒有名額了"| P4

    P1 -->|"post() → 剩 1"| P4_OK["Process 4 可以繼續了"]

    style SEM fill:#e3f2fd
    style P4 fill:#ffcdd2
    style P4_OK fill:#c8e6c9
```

類比：停車場有 3 個車位。前三台車可以直接停，
第四台車要在門口等，直到有車開走騰出車位。

---

## Resolved Signal — 多驅動器信號

當多個 process 同時驅動同一根信號線時，需要「解析」規則：

```mermaid
flowchart TD
    D1["Driver 1<br/>輸出: 1"] --> RS["sc_signal_resolved<br/>解析邏輯"]
    D2["Driver 2<br/>輸出: Z (高阻抗)"] --> RS
    D3["Driver 3<br/>輸出: 1"] --> RS

    RS --> RESULT["解析結果: 1"]

    style RS fill:#e3f2fd
    style RESULT fill:#c8e6c9
```

### 四值邏輯的解析表

| Driver A | Driver B | 解析結果 |
|----------|----------|----------|
| 0 | 0 | 0 |
| 0 | 1 | X (衝突!) |
| 0 | Z | 0 |
| 1 | 1 | 1 |
| 1 | Z | 1 |
| Z | Z | Z |

普通的 `sc_signal` 只允許一個寫入者（writer policy），
`sc_signal_resolved` 允許多個寫入者但會進行邏輯解析。

---

## 通訊元件如何對應到硬體

```mermaid
graph LR
    subgraph "SystemC 通訊元件"
        SIG["sc_signal"]
        FIFO_SC["sc_fifo"]
        MUX_SC["sc_mutex"]
        SEM_SC["sc_semaphore"]
        RES["sc_signal_resolved"]
    end

    subgraph "硬體對應"
        WIRE["導線 (Wire)"]
        BUFFER["緩衝器 / FIFO"]
        ARB["仲裁器 (Arbiter)"]
        RESOURCE["共享資源控制器"]
        BUS["三態匯流排"]
    end

    SIG -->|對應| WIRE
    FIFO_SC -->|對應| BUFFER
    MUX_SC -->|對應| ARB
    SEM_SC -->|對應| RESOURCE
    RES -->|對應| BUS

    style SIG fill:#e3f2fd
    style FIFO_SC fill:#e3f2fd
    style MUX_SC fill:#e3f2fd
    style SEM_SC fill:#e3f2fd
    style RES fill:#e3f2fd
    style WIRE fill:#fff3e0
    style BUFFER fill:#fff3e0
    style ARB fill:#fff3e0
    style RESOURCE fill:#fff3e0
    style BUS fill:#fff3e0
```

---

## 完整的通訊架構圖

```mermaid
classDiagram
    class sc_interface {
        <<abstract>>
    }

    class sc_signal_in_if~T~ {
        <<abstract>>
        +read() T
    }

    class sc_signal_inout_if~T~ {
        <<abstract>>
        +write(T)
    }

    class sc_fifo_in_if~T~ {
        <<abstract>>
        +read(T)
        +nb_read(T) bool
        +num_available() int
    }

    class sc_fifo_out_if~T~ {
        <<abstract>>
        +write(T)
        +nb_write(T) bool
        +num_free() int
    }

    class sc_mutex_if {
        <<abstract>>
        +lock() int
        +trylock() int
        +unlock() int
    }

    class sc_semaphore_if {
        <<abstract>>
        +wait() int
        +trywait() int
        +post() int
        +get_value() int
    }

    sc_interface <|-- sc_signal_in_if
    sc_signal_in_if <|-- sc_signal_inout_if
    sc_interface <|-- sc_fifo_in_if
    sc_interface <|-- sc_fifo_out_if
    sc_interface <|-- sc_mutex_if
    sc_interface <|-- sc_semaphore_if

    class sc_signal~T~ {
        +read() T
        +write(T)
    }
    class sc_fifo~T~ {
        +read(T)
        +write(T)
    }
    class sc_mutex {
        +lock()
        +unlock()
    }
    class sc_semaphore {
        +wait()
        +post()
    }

    sc_signal_inout_if <|.. sc_signal
    sc_fifo_in_if <|.. sc_fifo
    sc_fifo_out_if <|.. sc_fifo
    sc_mutex_if <|.. sc_mutex
    sc_semaphore_if <|.. sc_semaphore
```

---

## 相關模組

| 概念 | 文件 | 關係 |
|------|------|------|
| 模組階層 | [hierarchy.md](hierarchy.md) | Port 和 Export 是模組的對外介面 |
| 事件機制 | [events.md](events.md) | Channel 透過事件通知值的改變 |
| 排程機制 | [scheduling.md](scheduling.md) | request_update/update 是排程的核心部分 |
| 資料型別 | [datatypes.md](datatypes.md) | Signal 的模板參數決定傳輸什麼資料 |
| TLM | [tlm.md](tlm.md) | TLM 是更高層次的通訊抽象 |

### 對應的底層程式碼文件

| 原始碼概念 | 程式碼文件 |
|-----------|-----------|
| sc_interface | [doc_v2/code/sysc/communication/sc_interface.md](../code/sysc/communication/sc_interface.md) |
| sc_signal | [doc_v2/code/sysc/communication/sc_signal.md](../code/sysc/communication/sc_signal.md) |
| sc_signal_ifs | [doc_v2/code/sysc/communication/sc_signal_ifs.md](../code/sysc/communication/sc_signal_ifs.md) |
| sc_prim_channel | [doc_v2/code/sysc/communication/sc_prim_channel.md](../code/sysc/communication/sc_prim_channel.md) |
| sc_port | [doc_v2/code/sysc/communication/sc_port.md](../code/sysc/communication/sc_port.md) |
| sc_export | [doc_v2/code/sysc/communication/sc_export.md](../code/sysc/communication/sc_export.md) |
| sc_fifo | [doc_v2/code/sysc/communication/sc_fifo.md](../code/sysc/communication/sc_fifo.md) |
| sc_mutex | [doc_v2/code/sysc/communication/sc_mutex.md](../code/sysc/communication/sc_mutex.md) |
| sc_semaphore | [doc_v2/code/sysc/communication/sc_semaphore.md](../code/sysc/communication/sc_semaphore.md) |
| sc_signal_resolved | [doc_v2/code/sysc/communication/sc_signal_resolved.md](../code/sysc/communication/sc_signal_resolved.md) |

---

## 學習小提示

1. **Interface-Channel-Port 是 SystemC 最重要的設計模式**——理解它，就理解了一半的 SystemC
2. **Signal 的值不是立即更新的**——寫入後要到下一個 delta cycle 才生效，這是最常見的初學者困惑
3. **FIFO 會阻塞，Signal 不會**——選錯通訊元件會讓你的設計出現意想不到的行為
4. **普通 signal 只允許一個寫入者**——如果需要多個驅動器，用 `sc_signal_resolved`
5. **Mutex 和 Semaphore 主要用在抽象建模**——RTL 級的設計通常不用它們
6. **Port 的 `->` 運算子會被轉發到 Channel 的 Interface**——`port->read()` 其實是呼叫 Channel 的 `read()`
