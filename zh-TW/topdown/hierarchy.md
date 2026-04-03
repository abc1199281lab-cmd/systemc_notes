# 模組階層

## 生活類比：樂高積木組裝

SystemC 的模組階層就像用樂高積木蓋房子：

- **sc_object** = 每一顆樂高積木的基底——不管是牆壁、窗戶、門，都是樂高積木
- **sc_module** = 一個組裝好的子結構——比如「二樓的浴室」是一個模組
- **Port（埠口）** = 積木上的接頭——讓兩個子結構可以拼接在一起
- **Export（輸出口）** = 積木上的卡槽——接受別人的接頭
- **Channel（通道）** = 連接兩個接頭的管道——資料在這裡流動
- **物件樹** = 組裝說明書上的零件清單——每個零件有名字和歸屬

就像你可以把「浴室模組」拔出來換成另一個設計，
SystemC 的模組化設計也讓你可以替換和重用元件。

---

## sc_object：萬物之根

`sc_object` 是 SystemC 中所有具名物件的基礎類別。
所有模組、埠口、通道、process 都繼承自它。

```mermaid
classDiagram
    class sc_object {
        -m_name : string
        -m_parent : sc_object*
        -m_child_objects : vector
        +name() const char*
        +basename() const char*
        +get_parent_object() sc_object*
        +get_child_objects() vector
    }

    class sc_module {
        +SC_HAS_PROCESS()
        +SC_METHOD()
        +SC_THREAD()
        +SC_CTHREAD()
    }

    class sc_port_base {
        +bind()
        +operator()
    }

    class sc_export_base {
        +bind()
    }

    class sc_prim_channel {
        +request_update()
        +update()
    }

    class sc_process_b {
        +is_runnable()
        +trigger_static()
    }

    sc_object <|-- sc_module
    sc_object <|-- sc_port_base
    sc_object <|-- sc_export_base
    sc_object <|-- sc_prim_channel
    sc_object <|-- sc_process_b
```

### 階層式命名

每個 `sc_object` 都有一個**階層式名稱**，反映它在物件樹中的位置：

```
top                        # 頂層模組
top.cpu                    # top 裡面的 cpu 子模組
top.cpu.alu                # cpu 裡面的 alu 子模組
top.cpu.alu.port_a         # alu 的一個埠口
top.cpu.alu.method_p_0     # alu 的一個 process
```

```mermaid
graph TD
    TOP["top<br/>(sc_module)"]
    CPU["top.cpu<br/>(sc_module)"]
    MEM["top.memory<br/>(sc_module)"]
    ALU["top.cpu.alu<br/>(sc_module)"]
    REG["top.cpu.regs<br/>(sc_module)"]
    PA["top.cpu.alu.port_a<br/>(sc_in)"]
    PB["top.cpu.alu.port_b<br/>(sc_in)"]
    POUT["top.cpu.alu.port_out<br/>(sc_out)"]
    PROC["top.cpu.alu.compute<br/>(SC_METHOD)"]

    TOP --> CPU
    TOP --> MEM
    CPU --> ALU
    CPU --> REG
    ALU --> PA
    ALU --> PB
    ALU --> POUT
    ALU --> PROC

    style TOP fill:#e3f2fd
    style CPU fill:#e3f2fd
    style MEM fill:#e3f2fd
    style ALU fill:#e3f2fd
    style REG fill:#e3f2fd
    style PA fill:#c8e6c9
    style PB fill:#c8e6c9
    style POUT fill:#fff3e0
    style PROC fill:#fce4ec
```

---

## sc_module：設計的基本單元

`sc_module` 是你在 SystemC 中定義硬體元件的基本方式。
每個模組可以包含：

1. **子模組** — 更小的元件
2. **埠口（Port）** — 對外的介面
3. **Process** — 行為描述（SC_METHOD, SC_THREAD, SC_CTHREAD）
4. **內部信號** — 子模組之間的連線
5. **內部變數** — 模組的私有狀態

```cpp
SC_MODULE(ALU) {
    // Ports
    sc_in<int>  a, b;
    sc_in<int>  op;
    sc_out<int> result;

    // Process
    void compute() {
        if (op.read() == 0)
            result.write(a.read() + b.read());
        else
            result.write(a.read() - b.read());
    }

    SC_CTOR(ALU) {
        SC_METHOD(compute);
        sensitive << a << b << op;
    }
};
```

### 模組的建構流程

```mermaid
sequenceDiagram
    participant Main as sc_main
    participant MN as sc_module_name
    participant MOD as MyModule
    participant CTX as sc_simcontext

    Main->>MN: 建立 sc_module_name("mod")
    Note over MN: 將名稱壓入堆疊
    Main->>MOD: new MyModule("mod")
    MOD->>CTX: 註冊到物件樹
    MOD->>MOD: 建立子模組
    MOD->>MOD: 建立 port/export
    MOD->>MOD: 註冊 SC_METHOD/SC_THREAD
    MOD->>MN: 解構 sc_module_name
    Note over MN: 將名稱彈出堆疊
```

---

## Port、Export 與 Channel

這三者構成了 SystemC 模組之間通訊的鐵三角：

```mermaid
flowchart LR
    subgraph "Module A"
        PROC_A["Process A"]
        PORT["sc_out<br/>(Port)"]
    end

    subgraph "Module B"
        EXPORT["sc_export<br/>(Export)"]
        PROC_B["Process B"]
    end

    SIGNAL["sc_signal<br/>(Channel)"]

    PROC_A -->|寫入| PORT
    PORT -->|綁定到| SIGNAL
    SIGNAL -->|綁定到| EXPORT
    EXPORT -->|讀取| PROC_B

    style PORT fill:#c8e6c9
    style EXPORT fill:#fff3e0
    style SIGNAL fill:#e3f2fd
```

### Port — 模組的「插頭」

Port 定義了模組需要什麼樣的外部連接：

| Port 類型 | 方向 | 類比 |
|-----------|------|------|
| `sc_in<T>` | 輸入 | USB 的資料輸入線 |
| `sc_out<T>` | 輸出 | USB 的資料輸出線 |
| `sc_inout<T>` | 雙向 | 雙向的 I2C 資料線 |

### Export — 模組的「插座」

Export 讓模組直接提供一個介面的實作，
不需要透過中間的 channel。

```mermaid
flowchart LR
    subgraph "Module A"
        PA["sc_port<IF>"]
    end

    subgraph "Module B"
        EB["sc_export<IF>"]
        IMPL["介面實作"]
        EB -.-> IMPL
    end

    PA -->|直接綁定| EB

    style PA fill:#c8e6c9
    style EB fill:#fff3e0
```

### Channel — 通訊的管道

Channel 是實現介面的具體元件，負責資料的傳輸和同步。

---

## 綁定（Binding）

綁定是在 elaboration 階段把 port、export、channel 連接起來的過程。

```mermaid
flowchart TD
    subgraph "正確的綁定方式"
        P1["Port → Signal"]
        P2["Port → Export"]
        P3["Port → Port (父模組到子模組)"]
        P4["Export → Export (子模組到父模組)"]
    end

    subgraph "語法"
        S1["module.port(signal)"]
        S2["module.port(other.export)"]
        S3["child.port(parent.port)"]
        S4["parent.export(child.export)"]
    end

    P1 --- S1
    P2 --- S2
    P3 --- S3
    P4 --- S4
```

### 階層式綁定

在多層模組中，port 可以「穿越」層次：

```mermaid
graph TD
    subgraph "Top"
        subgraph "Sub A"
            PA["port_a"]
        end
        SIG["signal_x"]
        subgraph "Sub B"
            PB["port_b"]
        end
    end

    PA -->|綁定| SIG
    PB -->|綁定| SIG

    style PA fill:#c8e6c9
    style PB fill:#c8e6c9
    style SIG fill:#e3f2fd
```

---

## Elaboration 階段做了什麼？

```mermaid
flowchart TD
    A[開始 Elaboration] --> B["建立所有模組實例<br/>（由外到內，遞迴建構）"]
    B --> C["建立所有 Port 和 Export"]
    C --> D["綁定 Port ↔ Channel ↔ Export"]
    D --> E["註冊所有 Process<br/>（SC_METHOD, SC_THREAD）"]
    E --> F["呼叫 before_end_of_elaboration()"]
    F --> G["呼叫 end_of_elaboration()"]
    G --> H["檢查所有 Port 都已綁定"]
    H --> I{有未綁定<br/>的 Port？}
    I -->|是| ERR[報錯並中止]
    I -->|否| J[Elaboration 完成<br/>進入 Simulation]

    style A fill:#c8e6c9
    style J fill:#c8e6c9
    style ERR fill:#ffcdd2
```

### sc_module_name 的巧妙設計

`sc_module_name` 利用建構子和解構子的配對，
自動管理模組的名稱堆疊：

```mermaid
sequenceDiagram
    participant Stack as 名稱堆疊

    Note over Stack: 堆疊為空

    Note over Stack: MyTop top("top")
    Stack->>Stack: push("top")
    Note over Stack: ["top"]

    Note over Stack: MyCPU cpu("cpu") (在 top 的建構子中)
    Stack->>Stack: push("cpu")
    Note over Stack: ["top", "cpu"]

    Note over Stack: cpu 建構完成
    Stack->>Stack: pop()
    Note over Stack: ["top"]

    Note over Stack: top 建構完成
    Stack->>Stack: pop()
    Note over Stack: []
```

---

## 如何對應到硬體區塊

```mermaid
graph TD
    subgraph "SystemC 模組階層"
        SC_TOP["sc_module: SoC"]
        SC_CPU["sc_module: CPU"]
        SC_MEM["sc_module: Memory"]
        SC_BUS["sc_module: Bus"]
        SC_ALU["sc_module: ALU"]
        SC_CTRL["sc_module: Controller"]

        SC_TOP --> SC_CPU
        SC_TOP --> SC_MEM
        SC_TOP --> SC_BUS
        SC_CPU --> SC_ALU
        SC_CPU --> SC_CTRL
    end

    subgraph "對應的硬體區塊"
        HW_SOC["晶片 (SoC)"]
        HW_CPU["處理器核心"]
        HW_MEM["記憶體控制器"]
        HW_BUS["系統匯流排"]
        HW_ALU["算術邏輯單元"]
        HW_CTRL["控制單元"]

        HW_SOC --> HW_CPU
        HW_SOC --> HW_MEM
        HW_SOC --> HW_BUS
        HW_CPU --> HW_ALU
        HW_CPU --> HW_CTRL
    end

    SC_TOP -.->|對應| HW_SOC
    SC_CPU -.->|對應| HW_CPU
    SC_MEM -.->|對應| HW_MEM

    style SC_TOP fill:#e3f2fd
    style SC_CPU fill:#e3f2fd
    style SC_MEM fill:#e3f2fd
    style HW_SOC fill:#fff3e0
    style HW_CPU fill:#fff3e0
    style HW_MEM fill:#fff3e0
```

在硬體設計中：
- **模組** 對應一個硬體功能區塊（IP Block）
- **Port** 對應晶片的接腳或區塊的輸入/輸出
- **Signal** 對應實體的導線（wire）
- **物件樹** 對應硬體的階層式設計圖（block diagram）

---

## sc_object_manager 與 sc_module_registry

這兩個類別在幕後管理所有物件和模組：

```mermaid
classDiagram
    class sc_object_manager {
        -m_object_table : 名稱→物件的對照表
        +find_object(name) sc_object*
        +insert_object(name, obj)
        +remove_object(name)
        +hierarchy_push(obj)
        +hierarchy_pop()
        +hierarchy_curr() sc_object*
    }

    class sc_module_registry {
        -m_module_vec : vector~sc_module~
        +insert(module)
        +remove(module)
        +all() vector
    }

    class sc_simcontext {
    }

    sc_simcontext --> sc_object_manager : 擁有
    sc_simcontext --> sc_module_registry : 擁有
```

---

## 相關模組

| 概念 | 文件 | 關係 |
|------|------|------|
| 模擬引擎 | [simulation-engine.md](simulation-engine.md) | Elaboration 是模擬引擎生命週期的第一階段 |
| 通訊機制 | [communication.md](communication.md) | Port-Channel-Export 模式的詳細說明 |
| 事件機制 | [events.md](events.md) | Process 透過事件驅動執行 |
| 排程機制 | [scheduling.md](scheduling.md) | Process 的排程行為 |

### 對應的底層程式碼文件

| 原始碼概念 | 程式碼文件 |
|-----------|-----------|
| sc_object | [doc_v2/code/sysc/kernel/sc_object.md](../code/sysc/kernel/sc_object.md) |
| sc_module | [doc_v2/code/sysc/kernel/sc_module.md](../code/sysc/kernel/sc_module.md) |
| sc_module_name | [doc_v2/code/sysc/kernel/sc_module_name.md](../code/sysc/kernel/sc_module_name.md) |
| sc_module_registry | [doc_v2/code/sysc/kernel/sc_module_registry.md](../code/sysc/kernel/sc_module_registry.md) |
| sc_object_manager | [doc_v2/code/sysc/kernel/sc_object_manager.md](../code/sysc/kernel/sc_object_manager.md) |
| sc_port | [doc_v2/code/sysc/communication/sc_port.md](../code/sysc/communication/sc_port.md) |
| sc_export | [doc_v2/code/sysc/communication/sc_export.md](../code/sysc/communication/sc_export.md) |

---

## 學習小提示

1. **sc_object 就像 Java 的 Object 類別**——所有 SystemC 具名物件都繼承自它
2. **模組 = 容器**——它本身不「做事」，裡面的 process 才做事
3. **Port 定義「需要什麼」，Channel 提供「怎麼做」**——這是介面與實作分離的經典設計
4. **階層式名稱就是物件的地址**——用 `top.cpu.alu` 可以唯一識別任何物件
5. **Elaboration 完成後，結構就固定了**——模擬階段不能再新增或刪除模組
6. **畫物件樹是理解設計的第一步**——拿到一個 SystemC 設計，先畫出模組階層圖
