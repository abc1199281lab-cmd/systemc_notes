# 排程機制

## 生活類比：餐廳廚房的出餐系統

想像一間忙碌餐廳的廚房：

- **訂單** = 事件通知——「三號桌要牛排！」
- **出餐窗口** = Runnable Queue——所有準備好要出的菜排在這裡
- **廚師依序出菜** = Evaluate Phase——一道一道做完
- **服務生核對餐點** = Update Phase——確認所有菜都對了才端出去
- **一輪出餐完成** = 一個 Delta Cycle
- **等下一波訂單** = 推進模擬時間

所有同一輪的訂單是「同時到的」（同一個 delta），
但廚師只能一道一道做（單執行緒模擬），
所以 SystemC 必須有一套公平的排程規則。

---

## Evaluate-Update 典範

SystemC 排程的核心就是不斷重複兩個階段：

```mermaid
flowchart LR
    E["Evaluate 階段<br/>執行 process"] --> U["Update 階段<br/>更新 channel"]
    U --> E2["Evaluate 階段<br/>執行被喚醒的 process"]
    E2 --> U2["Update 階段<br/>再次更新"]
    U2 --> NEXT["...重複直到穩定..."]

    style E fill:#e3f2fd
    style U fill:#fff3e0
    style E2 fill:#e3f2fd
    style U2 fill:#fff3e0
```

### 為什麼需要分兩階段？

因為硬體的特性：**所有正反器在時鐘邊沿同時翻轉**。

如果允許一個 process 寫入值後，另一個 process 立刻就看到新值，
那麼執行順序就會影響結果——這不符合硬體行為。

分離成 Evaluate 和 Update 保證了：
1. 在 Evaluate 階段，所有 process 讀到的是**上一輪 Update 的結果**
2. 寫入的新值先「暫存」起來
3. 等所有 process 都跑完了，再統一 Update

---

## Delta Cycle 完整解析

一個 delta cycle 由一次 Evaluate + 一次 Update 組成：

```mermaid
flowchart TD
    subgraph "Delta Cycle #0"
        E0[/"Evaluate Phase<br/>從 runnable queue 取出 process<br/>依序執行"/]
        U0[/"Update Phase<br/>所有 request_update 的 channel<br/>統一更新值"/]
        E0 --> U0
    end

    subgraph "Delta Cycle #1"
        E1[/"Evaluate Phase<br/>被 delta 通知喚醒的 process"/]
        U1[/"Update Phase<br/>更新"/]
        E1 --> U1
    end

    U0 -->|"channel 值改變<br/>觸發 delta 通知"| E1

    subgraph "時間推進"
        ADV[/"推進到下一個<br/>timed event 的時間"/]
    end

    U1 -->|"沒有更多 delta 通知"| ADV

    style E0 fill:#e3f2fd
    style U0 fill:#fff3e0
    style E1 fill:#e3f2fd
    style U1 fill:#fff3e0
    style ADV fill:#e8f5e9
```

### 一個具體例子

假設有一個簡單的組合邏輯鏈：A → B → C

```
signal_a 改變 → process_b 讀 a 算 b → signal_b 改變 → process_c 讀 b 算 c
```

```mermaid
sequenceDiagram
    participant SA as signal_a
    participant PB as process_b
    participant SB as signal_b
    participant PC as process_c
    participant SC2 as signal_c

    Note over SA, SC2: 時間 = 10ns

    Note over SA, SC2: Delta #0
    Note over SA: signal_a 被外部改為 1
    SA->>PB: 值改變通知
    PB->>PB: 讀 signal_a = 1, 計算
    PB->>SB: 寫入 signal_b (request_update)

    Note over SA, SC2: Update
    SB->>SB: 值從 0 更新為 1

    Note over SA, SC2: Delta #1
    SB->>PC: 值改變通知
    PC->>PC: 讀 signal_b = 1, 計算
    PC->>SC2: 寫入 signal_c (request_update)

    Note over SA, SC2: Update
    SC2->>SC2: 值更新

    Note over SA, SC2: Delta #2 (無更多通知, delta cycle 結束)
    Note over SA, SC2: 推進時間到下一個事件
```

---

## Process 執行順序

### 關鍵觀念：同一個 delta 內，process 的執行順序是**不確定的**

```mermaid
graph TD
    Q[Runnable Queue<br/>包含 Process A, B, C] --> NOTE["引擎可能以任何順序<br/>執行 A, B, C"]
    NOTE --> O1["順序 1: A → B → C"]
    NOTE --> O2["順序 2: B → A → C"]
    NOTE --> O3["順序 3: C → B → A"]
    NOTE --> OX["... 其他排列 ..."]

    O1 --> RES["但因為 Evaluate-Update 分離<br/>不管什麼順序<br/>結果都一樣！"]
    O2 --> RES
    O3 --> RES
    OX --> RES

    style RES fill:#c8e6c9
    style Q fill:#e3f2fd
```

**這就是 Evaluate-Update 設計的精妙之處**：
因為所有 process 在 Evaluate 階段讀到的都是舊值，
寫入的新值到 Update 才生效，
所以不管引擎先執行哪個 process，最終結果都相同。

### SC_METHOD vs SC_THREAD 的排程差異

```mermaid
flowchart TD
    subgraph "SC_METHOD 的生命週期"
        M1[被事件喚醒] --> M2[從頭執行到尾]
        M2 --> M3[return]
        M3 --> M4[等待下一次事件]
        M4 --> M1
    end

    subgraph "SC_THREAD 的生命週期"
        T1[第一次啟動] --> T2[執行到 wait]
        T2 --> T3[暫停, 保存狀態]
        T3 --> T4[被事件喚醒]
        T4 --> T5[從 wait 之後繼續]
        T5 --> T2
    end

    style M1 fill:#e3f2fd
    style T1 fill:#fff3e0
```

---

## Runnable Queue 的運作

Runnable Queue 是排程器的核心資料結構，
儲存所有「準備好要執行」的 process：

```mermaid
classDiagram
    class sc_runnable {
        -m_methods_push_head : sc_method_process*
        -m_methods_pop : sc_method_process*
        -m_threads_push_head : sc_thread_process*
        -m_threads_pop : sc_thread_process*
        +init()
        +is_empty() bool
        +is_initialized() bool
        +push_back_method(method)
        +push_back_thread(thread)
        +pop_method() sc_method_process*
        +pop_thread() sc_thread_process*
        +remove_method(method)
        +remove_thread(thread)
    }

    note for sc_runnable "使用兩個指標實現<br/>push 和 pop 分離<br/>避免在 evaluate 階段<br/>新加入的 process 影響當前輪次"
```

### Push / Pop 分離機制

```mermaid
flowchart TD
    subgraph "Evaluate 階段"
        POP["從 Pop 端取出 process<br/>依序執行"]
        PUSH["新被喚醒的 process<br/>加入 Push 端"]
    end

    subgraph "兩輪之間"
        SWAP["Pop 端清空後<br/>Push 端變成下一輪的 Pop 端"]
    end

    POP --> SWAP
    PUSH --> SWAP
    SWAP --> POP

    style POP fill:#e3f2fd
    style PUSH fill:#fff3e0
    style SWAP fill:#e8f5e9
```

這個設計確保：在 evaluate 階段因為立即通知而被喚醒的 process，
會在**同一個 delta cycle** 內被執行，但不會打亂當前正在執行的順序。

---

## Timed vs Untimed 通知的排程

```mermaid
flowchart TD
    EVENT["事件通知"] --> TYPE{通知類型？}

    TYPE -->|"notify()"| IMM["立即通知<br/>加入當前 delta 的<br/>runnable queue"]
    TYPE -->|"notify(SC_ZERO_TIME)"| DELTA["Delta 通知<br/>加入下一個 delta 的<br/>runnable queue"]
    TYPE -->|"notify(t)"| TIMED["定時通知<br/>加入 timed event queue<br/>按時間排序"]

    IMM --> EVAL_NOW["在當前 evaluate<br/>階段內執行"]
    DELTA --> EVAL_NEXT["在下一個 evaluate<br/>階段執行"]
    TIMED --> TIME_ADV["等時間推進<br/>到達後執行"]

    style IMM fill:#ffcdd2
    style DELTA fill:#fff3e0
    style TIMED fill:#e3f2fd
```

### Timed Event Queue

定時事件使用優先佇列（priority queue）管理，按觸發時間排序：

```mermaid
graph TD
    subgraph "Timed Event Queue (Priority Queue)"
        T1["t=10ns: 時鐘上升沿"]
        T2["t=15ns: 資料到達"]
        T3["t=20ns: 時鐘下降沿"]
        T4["t=20ns: 超時計時器"]
    end

    T1 --> T2 --> T3 --> T4

    CURRENT["目前模擬時間 t=5ns"]
    CURRENT -.->|"推進到"| T1

    style CURRENT fill:#c8e6c9
    style T1 fill:#ffcdd2
```

當所有 delta cycle 都穩定了（沒有更多 delta 通知），
引擎才會推進模擬時間到 timed event queue 中的下一個時間點。

---

## 完整排程演算法

```mermaid
flowchart TD
    START([開始]) --> INIT["初始化<br/>所有 process 加入 runnable queue"]
    INIT --> EVAL

    EVAL["EVALUATE<br/>從 runnable queue 取出一個 process 執行"]
    EVAL --> MORE_P{runnable queue<br/>還有 process？}
    MORE_P -->|是| EVAL

    MORE_P -->|否| UPDATE["UPDATE<br/>更新所有 request_update 的 channel"]
    UPDATE --> DELTA_N{有 delta<br/>通知？}

    DELTA_N -->|是| ADD_DELTA["把被通知的 process<br/>加入 runnable queue"]
    ADD_DELTA --> EVAL

    DELTA_N -->|否| TNOTIFY{"有 timed<br/>通知？<br/>且未超過<br/>指定時間？"}

    TNOTIFY -->|是| ADVANCE["推進模擬時間到<br/>最近的 timed event"]
    ADVANCE --> FIRE["觸發該時間的所有事件<br/>相關 process 加入 runnable queue"]
    FIRE --> EVAL

    TNOTIFY -->|否| END([模擬結束])

    style START fill:#c8e6c9
    style END fill:#ffcdd2
    style EVAL fill:#e3f2fd
    style UPDATE fill:#fff3e0
    style ADVANCE fill:#e8f5e9
```

---

## 常見排程陷阱

### 陷阱一：無限 Delta Cycle

```cpp
// 危險！A 和 B 互相觸發，永遠停不下來
SC_METHOD(process_a);
sensitive << sig_b;
void process_a() { sig_a.write(!sig_b.read()); }

SC_METHOD(process_b);
sensitive << sig_a;
void process_b() { sig_b.write(!sig_a.read()); }
```

```mermaid
flowchart LR
    A["Process A<br/>sig_a = !sig_b"] -->|"sig_a 改變"| B["Process B<br/>sig_b = !sig_a"]
    B -->|"sig_b 改變"| A

    style A fill:#ffcdd2
    style B fill:#ffcdd2
```

SystemC 引擎會在偵測到過多 delta cycle 時報錯並中止。

### 陷阱二：SC_METHOD 中呼叫 wait()

```cpp
SC_METHOD(my_method);
void my_method() {
    wait(10, SC_NS);  // 錯誤！SC_METHOD 不能 wait!
}
```

SC_METHOD 沒有自己的執行堆疊，無法暫停和恢復。
只有 SC_THREAD 和 SC_CTHREAD 才能呼叫 `wait()`。

---

## 相關模組

| 概念 | 文件 | 關係 |
|------|------|------|
| 模擬引擎 | [simulation-engine.md](simulation-engine.md) | 排程器是模擬引擎的心臟 |
| 事件機制 | [events.md](events.md) | 事件觸發排程 |
| 通訊機制 | [communication.md](communication.md) | Channel 的 request_update/update 是排程的一部分 |
| 模組階層 | [hierarchy.md](hierarchy.md) | Process 定義在模組中 |

### 對應的底層程式碼文件

| 原始碼概念 | 程式碼文件 |
|-----------|-----------|
| sc_simcontext | [doc_v2/code/sysc/kernel/sc_simcontext.md](../code/sysc/kernel/sc_simcontext.md) |
| sc_runnable | [doc_v2/code/sysc/kernel/sc_runnable.md](../code/sysc/kernel/sc_runnable.md) |
| sc_method_process | [doc_v2/code/sysc/kernel/sc_method_process.md](../code/sysc/kernel/sc_method_process.md) |
| sc_thread_process | [doc_v2/code/sysc/kernel/sc_thread_process.md](../code/sysc/kernel/sc_thread_process.md) |
| sc_event | [doc_v2/code/sysc/kernel/sc_event.md](../code/sysc/kernel/sc_event.md) |
| sc_prim_channel | [doc_v2/code/sysc/communication/sc_prim_channel.md](../code/sysc/communication/sc_prim_channel.md) |

---

## 學習小提示

1. **Evaluate-Update 是排程的靈魂**——理解了這個，就理解了 SystemC 為什麼能正確模擬硬體
2. **同一 delta 內的執行順序不確定**——永遠不要依賴 process 的執行順序來寫正確的程式碼
3. **Delta cycle 很快但不是免費的**——太多 delta cycle（如組合邏輯鏈太長）會影響模擬速度
4. **時間只在 delta cycle 穩定後才推進**——模擬時間是「離散跳躍」的，不是連續流動的
5. **初學者先掌握 SC_THREAD + wait()**——比 SC_METHOD + 靜態敏感度更直覺
6. **畫時序圖來除錯**——遇到排程問題時，手動畫出每個 delta cycle 中各 process 看到的值
