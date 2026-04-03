# stage1 -- 加減法運算級

> **檔案**: `stage1.h`, `stage1.cpp` | **角色**: 管線第一級

## 軟體類比

`stage1` 就像管線中的第一個 **transform** 步驟。它讀取兩個輸入值，產生兩個輸出值（sum 和 diff）。

```python
# Python 類比
def stage1_transform(a, b):
    return a + b, a - b
```

在 Unix pipe 中，它相當於一個 `awk` 指令，讀取輸入、做運算、輸出結果。

## 模組結構

### Header（stage1.h）

```cpp
SC_MODULE(stage1) {
    sc_in<bool>    clk;    // clock 輸入
    sc_in<double>  in1;    // 輸入 a（來自 numgen.out1）
    sc_in<double>  in2;    // 輸入 b（來自 numgen.out2）
    sc_out<double> sum;    // 輸出 a + b
    sc_out<double> diff;   // 輸出 a - b

    void entry();

    SC_CTOR(stage1) {
        SC_METHOD(entry);
        dont_initialize();
        sensitive << clk.pos();
    }
};
```

### Implementation（stage1.cpp）

```cpp
void stage1::entry() {
    double a = in1.read();   // 從輸入 port 讀取
    double b = in2.read();
    sum.write(a + b);        // 寫入輸出 port
    diff.write(a - b);
}
```

## 關鍵概念

### sc_in 和 sc_out -- 模組的對外介面

`sc_in<T>` 和 `sc_out<T>` 是 SystemC 的 port 類型，定義模組的輸入和輸出介面。

它們的軟體類比：

| SystemC | 軟體類比 |
|---|---|
| `sc_in<double>` | 函式參數（`const double& input`） |
| `sc_out<double>` | 函式回傳值（`double& output`） |
| `sc_inout<double>` | 可讀可寫的參考（`double& ref`） |

### read() 和 write() 方法

Port 不能直接用 `=` 賦值，必須透過 `read()` 和 `write()` 方法存取：

```cpp
// 正確
double a = in1.read();
sum.write(a + b);

// 錯誤 -- 不能直接存取
// double a = in1;       // 編譯錯誤
// sum = a + b;          // 編譯錯誤
```

為什麼要這樣設計？因為 `read()` 和 `write()` 背後可能涉及 delta cycle 的同步機制。直接賦值無法保證模擬的正確性。

這跟 React 的 `useState` 理念類似：你不能直接修改 state，必須透過 `setState()` 更新，因為框架需要追蹤變化。

### Port 與 Signal 的關係

Port 本身不儲存資料，它必須**綁定（bind）**到一個 `sc_signal` 才能運作。`sc_signal` 才是真正儲存值的地方。

```
sc_out<double> sum  ──綁定到──>  sc_signal<double> sum_sig  <──綁定到──  sc_in<double> sum
（stage1 的輸出）                  （main.cpp 宣告）                     （stage2 的輸入）
```

這就像 Unix pipe：`|` 符號建立了一個匿名管道，連接左邊 process 的 stdout 和右邊 process 的 stdin。

## 運算邏輯

以第一個 clock 為例：

- 輸入：`a = 134.56`, `b = 98.24`
- 輸出：`sum = 232.80`, `diff = 36.32`

注意：由於管線的 delta cycle 延遲，stage1 讀到的值實際上是 numgen 在**前一個** delta cycle 寫入的值。這是 `sc_signal` 的特性 -- 寫入的值要到下一個 delta cycle 才能被讀取。
