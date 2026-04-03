# sc_stub.h - 存根通道、未綁定與固定值連接

## 概觀

這個檔案提供了三個重要的工具，用於處理模組埠「不需要實際連接」的場景：

1. **`sc_stub<T>`** - 一個什麼都不做的假通道（存根通道）
2. **`sc_unbound`** - 全域物件，用於標記「這個埠故意不連接」
3. **`sc_tie::value(val)`** - 函式，用於把埠綁定到一個固定值

這些工具在 2021 年加入 SystemC，解決了長期以來埠必須全部綁定的痛點。

## 核心概念 / 生活化比喻

### 電路板上的空腳位

想像一塊主機板上的晶片：

- 有些腳位**用不到**但不能讓它飄著（浮接），否則可能產生不確定行為
- **sc_unbound** = 把腳位接到一個空的端子（不接任何東西，但有明確標記「故意不用」）
- **sc_tie::value(0)** = 把腳位直接焊到地線（固定為 0）
- **sc_tie::value(1)** = 把腳位直接焊到電源（固定為 1）

### 為何需要這個？

在大型 SoC 設計中，一個 IP 模組可能有 100 個埠，但在特定整合中只用到 30 個。以前你必須為每個不用的埠手動建立一個 `sc_signal` 並連接，非常繁瑣。現在只需要：

```cpp
module.unused_port(sc_unbound);           // 故意不連接
module.config_port(sc_tie::value(0x42));  // 綁定到固定值
```

## 類別詳細說明

### `sc_stub<T>` - 存根通道

```cpp
template <typename T>
class sc_stub : public sc_signal_inout_if<T>, public sc_prim_channel
```

一個「假」通道，實作了完整的信號介面但什麼都不做：

| 操作 | 行為 |
|------|------|
| `read()` | 回傳初始值 `m_init_val` |
| `write(val)` | **忽略**，什麼都不做 |
| `default_event()` | 回傳一個永遠不會觸發的空事件 |
| `value_changed_event()` | 同上 |
| `event()` | 永遠回傳 `false` |
| `posedge()` / `negedge()` | 永遠回傳 `false` |

### `sc_stub_registry` - 存根管理中心

```cpp
class sc_stub_registry
{
    void insert(sc_prim_channel* stub_);
    void clear();
};
```

所有動態建立的 `sc_stub` 都會註冊到這裡，由 `sc_simcontext` 管理生命週期。這避免了記憶體洩漏——因為 `sc_unbound` 和 `sc_tie::value()` 會用 `new` 建立 `sc_stub`，需要有人負責刪除。

安全檢查：
- 不能在**模擬執行中**建立 stub
- 不能在 **elaboration 完成後**建立 stub

### `sc_unbound` - 未綁定標記

```cpp
static sc_unbound_impl const sc_unbound = {};
```

一個全域靜態物件。它的魔法在於型別轉換運算子：

```cpp
template <typename T>
operator sc_signal_inout_if<T>&() const
{
    sc_stub<T>* stub = new sc_stub<T>(sc_gen_unique_name("sc_unbound"));
    sc_get_curr_simcontext()->get_stub_registry()->insert(...);
    return *stub;
}
```

當你寫 `port(sc_unbound)` 時：
1. 編譯器自動推導出 `T` 的型別
2. 建立一個新的 `sc_stub<T>`
3. 註冊到 stub registry
4. 回傳介面參考，完成綁定

**限制**：只能用於 `sc_inout` 或 `sc_out` 類型的埠（輸出/雙向），不能用於純輸入埠 `sc_in`，因為讀取值是未定義的。

### `sc_tie::value()` - 固定值綁定

```cpp
namespace sc_tie {
    template <typename T>
    sc_signal_in_if<T>& value(const T& val);
}
```

```cpp
sc_stub<T>* stub = new sc_stub<T>(sc_gen_unique_name("sc_tie::value"), val);
```

與 `sc_unbound` 類似，但：
- 可以指定一個固定值
- 回傳的是 `sc_signal_in_if<T>&`，所以**可以用於 `sc_in` 埠**
- 每次呼叫都建立一個新的 stub

```mermaid
graph TD
    subgraph "sc_unbound 使用流程"
        UB["port(sc_unbound)"] --> TC["型別轉換<br/>operator sc_signal_inout_if<T>&()"]
        TC --> NEW1["new sc_stub<T>(\"sc_unbound_0\")"]
        NEW1 --> REG1["註冊到 stub_registry"]
        REG1 --> BIND1["綁定完成"]
    end
    subgraph "sc_tie::value 使用流程"
        TV["port(sc_tie::value(42))"] --> FN["sc_tie::value(42)"]
        FN --> NEW2["new sc_stub<T>(\"sc_tie::value_0\", 42)"]
        NEW2 --> REG2["註冊到 stub_registry"]
        REG2 --> BIND2["綁定完成<br/>read() 永遠回傳 42"]
    end
```

## 成員變數

### `sc_stub<T>`

| 變數 | 型別 | 說明 |
|------|------|------|
| `m_init_val` | `T` | 初始值/固定值，`read()` 會回傳這個值 |
| `ev` | `sc_event` | 永遠不會觸發的空事件 |

### `sc_stub_registry`

| 變數 | 型別 | 說明 |
|------|------|------|
| `m_stub_vec` | `std::vector<sc_prim_channel*>` | 所有已註冊的 stub |
| `m_simc` | `sc_simcontext*` | 所屬的模擬上下文 |

## 設計原理 / RTL 背景

### 硬體中的 tie-off

在 ASIC 設計中，「tie-off」是常見的做法：

- **Tie-high**：把不用的輸入接到 VDD（電源，邏輯 1）
- **Tie-low**：把不用的輸入接到 GND（地線，邏輯 0）
- **NC**（No Connect）：不連接（但需要明確標示）

`sc_tie::value()` 和 `sc_unbound` 正是模擬了這些硬體設計實務。

### 命名慣例

- `sc_unbound` 建立的 stub 命名為 `sc_unbound_0`, `sc_unbound_1`, ...
- `sc_tie::value()` 建立的 stub 命名為 `sc_tie::value_0`, `sc_tie::value_1`, ...

使用 `sc_gen_unique_name` 確保每個 stub 都有唯一名稱。

### 記憶體管理

所有 stub 由 `sc_stub_registry` 持有，在 `sc_simcontext` 解構時透過 `clear()` 一次性刪除。這是一種簡單的「Arena 分配」模式。

## 相關檔案

- `sc_signal_ifs.h` - 信號介面定義（`sc_signal_in_if`, `sc_signal_inout_if`）
- `sc_prim_channel.h` - 原始通道基礎類別
- `sc_simcontext.h` - 模擬上下文（管理 stub registry）
- `sc_communication_ids.h` - 錯誤訊息 ID
