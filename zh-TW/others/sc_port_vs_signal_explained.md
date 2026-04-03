# SystemC：sc_in/out/signal 與 sc_channel/sc_interface/sc_port 的差異

> 以 `examples/sysc/pipe` 與 `examples/sysc/simple_fifo` 為例

---

## 兩種設計風格

### `pipe` — RTL / 信號風格

```
sc_signal<double>  ←→  sc_in<double> / sc_out<double>
  （通道）                      （埠）
```

- `sc_signal<T>` **本身就是通道** — 負責儲存值，並在值改變時發出通知
- `sc_in<T>` / `sc_out<T>` 是**預建的埠**，只懂得對信號做 `read()` / `write()`
- 模組由**時脈驅動**（`SC_METHOD` + `sensitive << clk.pos()`）
- 通訊模型：**RTL delta-cycle 握手** — 本週期寫入，下一個 delta 週期讀取

```cpp
// pipe/main.cpp:49-58 — 宣告連線（wire）
sc_signal<double> in1, in2, sum;

// pipe/stage1.h:42-46 — 預建的型別化埠
sc_in<double>  in1;
sc_out<double> sum;

// 綁定：埠 ← 信號
S1.in1(in1);   // main.cpp:64
```

---

### `simple_fifo` — TLM / 通道風格

```
sc_interface  ←  自訂介面（write_if / read_if）
     ↓
sc_channel    ←  fifo（實作兩個介面）
     ↑
sc_port<IF>   ←  producer::out、consumer::in
```

三個角色**刻意分離**：

| 角色 | 類別 | 職責 |
|---|---|---|
| `sc_interface` | `write_if`、`read_if` | 宣告 API 契約（純虛擬函式） |
| `sc_channel` | `fifo` | 實作 API，擁有資料與事件 |
| `sc_port<IF>` | `producer::out` | 接受任何實作 `IF` 之通道的插座 |

```cpp
// simple_fifo.cpp:45-57 — 定義介面契約
class write_if : virtual public sc_interface {
    virtual void write(char) = 0;
    virtual void reset() = 0;
};

// simple_fifo.cpp:59 — 通道同時實作兩個介面
class fifo : public sc_channel, public write_if, public read_if { ... }

// simple_fifo.cpp:97, 118 — 埠以介面為型別，而非資料型別
sc_port<write_if> out;   // producer
sc_port<read_if>  in;    // consumer

// 綁定：埠 ← 通道物件
prod_inst->out(*fifo_inst);  // simple_fifo.cpp:154
```

---

## 核心概念差異

```
pipe（RTL 風格）：
  模組 → sc_out<T> → sc_signal<T> → sc_in<T> → 模組
           （埠）       （通道）        （埠）
        由相同型別 T 緊密綁定，只有一種線模型

simple_fifo（TLM 風格）：
  模組 → sc_port<write_if> ────── fifo ────── sc_port<read_if> → 模組
              （埠）              （通道）            （埠）
         以介面鬆散耦合；可替換 fifo 實作，不需改動模組
```

**核心洞見：**

- `sc_in/out` + `sc_signal` = 被資料型別 `T` **緊密耦合**，埠與通道認識彼此的型別
- `sc_port<IF>` + `sc_channel` = 被介面**鬆散耦合**，可隨時換掉通道的實作，producer / consumer 完全不受影響

> `sc_in<T>` 底層實際上就是 `sc_port<sc_signal_in_if<T>>` 的便利簡寫，
> 是針對信號特定情境的語法糖。

---

## 三個角色的職責總表

| 角色 | RTL 風格 | TLM 風格 |
|---|---|---|
| **介面（Interface）** | 隱含在 `sc_signal_in_if<T>` 內 | 自訂 `write_if` / `read_if` |
| **通道（Channel）** | `sc_signal<T>`（內建） | 自訂 `fifo`（繼承 `sc_channel`） |
| **埠（Port）** | `sc_in<T>` / `sc_out<T>` | `sc_port<write_if>` / `sc_port<read_if>` |
| **驅動方式** | 時脈邊緣觸發（`SC_METHOD`） | 阻塞式呼叫（`SC_THREAD` + `wait`） |
| **耦合程度** | 緊密（型別綁定） | 鬆散（介面綁定） |
| **適用層次** | RTL / 時序精確 | TLM / 非時序或鬆散時序 |

---

## 何時選用哪一種

| 情境 | 建議選用 |
|---|---|
| 時脈驅動的 RTL / 時序精確模型 | `sc_in/out` + `sc_signal` |
| 非時序 / 鬆散時序的 TLM 模型 | `sc_port<IF>` + 自訂 `sc_channel` |
| 想在不改動模組的前提下替換通道實作 | `sc_port<IF>` + `sc_channel` |
| 模組間只需傳遞簡單的值 | `sc_signal` |

---

## 類比小結

- `sc_signal` 就像 Verilog 的 `wire`，直接連兩個腳位
- `sc_port<IF>` + `sc_channel` 就像 C++ 的 **虛擬函式介面** — 呼叫者只認識介面，實作可以自由替換
