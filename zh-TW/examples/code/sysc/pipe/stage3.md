# stage3 -- 次方運算級

> **檔案**: `stage3.h`, `stage3.cpp` | **角色**: 管線第三級

## 軟體類比

`stage3` 是管線的最後一個運算階段，計算 `prod` 的 `quot` 次方。這是整條管線中計算量最大的一步。

```python
# Python 類比
def stage3_transform(prod, quot):
    if prod > 0 and quot > 0:
        return prod ** quot
    else:
        return 0.0
```

## 模組結構

### Header（stage3.h）

```cpp
SC_MODULE(stage3) {
    sc_in<bool>    clk;
    sc_in<double>  prod;   // 來自 stage2
    sc_in<double>  quot;   // 來自 stage2
    sc_out<double> powr;   // prod ^ quot

    void entry();

    SC_CTOR(stage3) {
        SC_METHOD(entry);
        dont_initialize();
        sensitive << clk.pos();
    }
};
```

### Implementation（stage3.cpp）

```cpp
void stage3::entry() {
    double a = prod.read();
    double b = quot.read();

    if (a > 0.0 && b > 0.0)
        powr.write(pow(a, b));
    else
        powr.write(0.0);      // 負值/零值防護
}
```

## 關鍵概念

### 為什麼需要正值防護？

`pow(base, exponent)` 函式在某些輸入下會產生問題：

| base | exponent | `pow()` 結果 | 問題 |
|---|---|---|---|
| 正數 | 正數 | 正常 | 無 |
| 0 | 正數 | 0 | 通常可接受 |
| 負數 | 非整數 | NaN | 數學上無定義（複數） |
| 正數 | 負數 | 正常（取倒數） | 可能溢出 |
| 0 | 0 | 1（按慣例） | 數學上有爭議 |

程式碼選擇最保守的策略：只有 **base > 0 且 exponent > 0** 時才計算，否則輸出 `0.0`。

這和 stage2 的除零防護理念一致 -- 在硬體設計中，寧可輸出安全的預設值，也不要讓 `NaN` 或 `Inf` 污染下游資料。

### 管線延遲的累積

到了 stage3，資料已經經過了三級管線。由於每一級都在 clock 正緣讀取並輸出，而 `sc_signal` 的值更新有一個 delta cycle 的延遲，stage3 看到的資料實際上是 **numgen 在幾個 clock 之前**產生的。

```
Clock 1: numgen 寫入 (a1, b1)
Clock 2: stage1 讀到 (a1, b1)，寫入 (sum1, diff1)
          numgen 寫入 (a2, b2)
Clock 3: stage2 讀到 (sum1, diff1)，寫入 (prod1, quot1)
          stage1 讀到 (a2, b2)，寫入 (sum2, diff2)
          numgen 寫入 (a3, b3)
Clock 4: stage3 讀到 (prod1, quot1)，寫入 (powr1)
          ...（以此類推）
```

這就是管線的核心概念：**延遲（latency）增加了**（第一筆結果要等 4 個 clock），但**吞吐量（throughput）不受影響**（之後每個 clock 都有一筆結果出來）。

## 運算範例

延續前面的數值（注意管線延遲）：

- 輸入：`prod = 8455.30`, `quot = 6.41`
- 兩者都 > 0，所以計算 `pow(8455.30, 6.41)`
- 結果是一個非常大的數值

由於 `prod` 的值一般很大，而 `quot` 也不小，`pow()` 的結果往往會非常大甚至溢出。這在範例中是預期行為 -- 這是一個教學範例，不是實際的數值計算系統。
