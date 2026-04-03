# Packet -- 封包資料結構

## 軟體類比

`pkt` 就是一個 **訊息物件（message object）**，包含 payload 和 routing metadata。類似：

```python
@dataclass
class Packet:
    data: int          # 8-bit payload（實際攜帶的資料）
    id: int            # 4-bit sender ID（誰送的）
    dest0: bool        # 是否送往 port 0
    dest1: bool        # 是否送往 port 1
    dest2: bool        # 是否送往 port 2
    dest3: bool        # 是否送往 port 3
```

或者用網路的語言來說，這就是一個簡化的 **Ethernet frame**：`data` 是 payload，`id` 是 source address，`dest0`..`dest3` 是 destination address（用 bitmask 表示，支援 multicast）。

## 結構定義

```cpp
struct pkt {
    sc_int<8> data;    // 8-bit 有號整數，封包的資料內容
    sc_int<4> id;      // 4-bit 有號整數，發送者 ID
    bool dest0;        // 目的地 0 旗標
    bool dest1;        // 目的地 1 旗標
    bool dest2;        // 目的地 2 旗標
    bool dest3;        // 目的地 3 旗標
};
```

### 欄位說明

| 欄位 | 型別 | 位元數 | 範圍 | 用途 |
|------|------|--------|------|------|
| `data` | `sc_int<8>` | 8 | -128 ~ 127 | 封包攜帶的資料（payload） |
| `id` | `sc_int<4>` | 4 | -8 ~ 7 | 發送者識別碼（實際只用 0-3） |
| `dest0` | `bool` | 1 | true/false | 是否送往 receiver 0 |
| `dest1` | `bool` | 1 | true/false | 是否送往 receiver 1 |
| `dest2` | `bool` | 1 | true/false | 是否送往 receiver 2 |
| `dest3` | `bool` | 1 | true/false | 是否送往 receiver 3 |

整個封包共 16 bit，對應 README 中的格式：

```
I<-ad(4)->I<-id(4)->I<---data(8)--->I
+-----------------------------------+
| | | | | | | | | | | | | | | | | | |
+-----------------------------------+
```

### 目的地 bitmask 範例

| dest3 | dest2 | dest1 | dest0 | 意義 |
|-------|-------|-------|-------|------|
| 0 | 0 | 0 | 1 | Unicast: 只送 receiver 0 |
| 0 | 1 | 1 | 0 | Multicast: 送 receiver 1 和 2 |
| 1 | 1 | 1 | 1 | Broadcast: 送所有 receiver |

Sender 產生的 dest 值範圍是 1-15（`rand()%15+1`），保證至少有一個目的地。

## 必要的 SystemC 支援函式

要讓自定義 struct 能用在 `sc_signal<pkt>` 中，SystemC 要求提供以下函式：

### `operator==`

```cpp
inline bool operator == (const pkt& rhs) const {
    return (rhs.data == data && rhs.id == id && ...);
}
```

`sc_signal` 需要比較新舊值來決定是否觸發事件。如果新值與舊值相同，就不觸發。

**軟體類比**：就像 React 的 `shouldComponentUpdate()` -- 框架需要知道值是否真的變了，來決定要不要通知監聽者。

### `operator<<`

```cpp
inline ostream& operator << (ostream& os, const pkt&) {
    os << "streaming of struct pkt not implemented";
    return os;
}
```

用於除錯輸出。這裡只是一個 placeholder，沒有真正印出封包內容。

### `sc_trace`

```cpp
void sc_trace(sc_trace_file* tf, const pkt& a, const std::string& name) {
    sc_trace(tf, a.data, name + ".data");
    sc_trace(tf, a.id, name + ".id");
    sc_trace(tf, a.dest0, name + ".dest0");
    // ...
}
```

用於波形追蹤（waveform tracing）。將 struct 的每個欄位分別寫入 trace 檔案，以便在波形檢視器中觀察。

**軟體類比**：就像 structured logging -- 把一個物件的每個欄位分別記錄到 log 中，而不是只記一個 `toString()` 的結果。

## 設計觀察

這個封包結構反映了硬體設計中「一切都是 bit」的思維：
- 目的地用 4 個獨立的 `bool` 而不是一個 `int`，因為硬體中每個 bit 可以獨立操作
- 使用 `sc_int<8>` 而不是 `int`，明確指定位元寬度
- 整個 struct 可以精確映射到 16 根實體線路（wire）
