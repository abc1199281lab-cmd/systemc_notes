# numgen -- 數值產生器

> **檔案**: `numgen.h`, `numgen.cpp` | **角色**: 資料來源（data source）

## 軟體類比

`numgen` 就像一個**感測器**或**資料產生器**，每個 clock tick 產生一組新的讀數。

在軟體世界中，這類似於：
- Python 的 generator function（每次 `yield` 一組值）
- Python 的 asyncio StreamReader（每次 `read` 一筆資料）
- Kafka producer（每個間隔送出一筆 event）

```python
# Python 類比
def numgen():
    a, b = 134.56, 98.24
    while True:
        yield a, b
        a -= 1.5
        b -= 2.8
```

## 模組結構

### Header（numgen.h）

```cpp
SC_MODULE(numgen) {
    sc_in<bool>   clk;    // clock 輸入
    sc_out<double> out1;   // 輸出 a
    sc_out<double> out2;   // 輸出 b

    void entry();          // 每個 clock 正緣呼叫

    SC_CTOR(numgen) {
        SC_METHOD(entry);           // 註冊為 SC_METHOD
        dont_initialize();          // 不要在模擬開始時自動呼叫
        sensitive << clk.pos();     // 只在 clock 正緣觸發
    }
};
```

### Implementation（numgen.cpp）

```cpp
void numgen::entry() {
    out1.write(a);   // 輸出當前的 a
    out2.write(b);   // 輸出當前的 b
    a -= 1.5;        // 遞減 a
    b -= 2.8;        // 遞減 b
}
```

## 關鍵概念

### SC_METHOD 的運作方式

`SC_METHOD(entry)` 告訴 SystemC：「每次指定的 sensitivity 事件發生時，呼叫 `entry()` 函式。」

這和 Python 的 callback 註冊非常相似：

```python
# Python 類比
def on_posedge():
    output1 = a
    output2 = b
    a -= 1.5
    b -= 2.8

clock.on_posedge(on_posedge)
```

重要限制：SC_METHOD **不能**呼叫 `wait()`。它必須在一次呼叫內完成所有工作並返回。如果你需要在中途暫停，請改用 SC_THREAD。

### dont_initialize() 的作用

預設情況下，SystemC 會在模擬開始時（時間 0）自動呼叫所有 process 一次，不管觸發條件是否滿足。`dont_initialize()` 阻止這個行為。

為什麼需要它？因為 `numgen` 應該只在 clock 正緣才輸出資料。如果不加 `dont_initialize()`，它會在時間 0 就寫出初始值，這可能不是我們想要的行為。

### sensitive << clk.pos()

`sensitive` 是 SystemC 的 sensitivity list，決定什麼事件會觸發這個 process。`clk.pos()` 表示「clock 信號的正緣（rising edge，從 0 變成 1）」。

這相當於硬體的：

```
// 硬體類比（Verilog）
always @(posedge clk) begin
    out1 <= a;
    out2 <= b;
end
```

### SC_METHOD vs SC_THREAD 比較

| | SC_METHOD | SC_THREAD |
|---|---|---|
| 類比 | callback / event handler | coroutine / Python coroutine (asyncio) |
| 執行 | 每次觸發從頭到尾 | 可用 `wait()` 暫停 |
| `wait()` | **禁止** | 允許 |
| 狀態 | 用成員變數保存（如 `a`, `b`） | 用 local 變數 + `wait()` |
| 效能 | 快（無 stack 切換） | 較慢（需保存/恢復 stack） |

本範例中 `numgen` 的邏輯很簡單（寫出、遞減），用 SC_METHOD 就夠了。如果需要複雜的多步驟流程（例如：先送 header，等回應，再送 payload），就適合用 SC_THREAD。

## 資料輸出序列

| Clock | out1 (a) | out2 (b) |
|---|---|---|
| 1 | 134.56 | 98.24 |
| 2 | 133.06 | 95.44 |
| 3 | 131.56 | 92.64 |
| 4 | 130.06 | 89.84 |
| ... | -1.5/cycle | -2.8/cycle |
