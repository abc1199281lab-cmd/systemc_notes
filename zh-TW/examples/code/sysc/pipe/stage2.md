# stage2 -- 乘除法運算級

> **檔案**: `stage2.h`, `stage2.cpp` | **角色**: 管線第二級

## 軟體類比

`stage2` 接收 stage1 的 sum 和 diff，計算它們的**乘積（product）**和**商（quotient）**。這就像 ETL 管線中的第二個 transform 步驟。

```python
# Python 類比
def stage2_transform(sum_val, diff_val):
    prod = sum_val * diff_val
    quot = sum_val / diff_val if diff_val != 0 else 5.0
    return prod, quot
```

## 模組結構

### Header（stage2.h）

```cpp
SC_MODULE(stage2) {
    sc_in<bool>    clk;
    sc_in<double>  sum;    // 來自 stage1
    sc_in<double>  diff;   // 來自 stage1
    sc_out<double> prod;   // sum * diff
    sc_out<double> quot;   // sum / diff

    void entry();

    SC_CTOR(stage2) {
        SC_METHOD(entry);
        dont_initialize();
        sensitive << clk.pos();
    }
};
```

### Implementation（stage2.cpp）

```cpp
void stage2::entry() {
    double a = sum.read();
    double b = diff.read();

    prod.write(a * b);

    if (b != 0.0)
        quot.write(a / b);
    else
        quot.write(5.0);   // 除零防護：使用預設值
}
```

## 關鍵概念

### 除零防護（Division-by-Zero Guard）

`stage2` 在做除法前檢查 `diff` 是否為零。如果為零，輸出一個安全的預設值 `5.0`，避免產生 `Inf` 或 `NaN`。

這是一個常見的**防禦式程式設計（defensive programming）**模式：

```python
# Python -- 類似的防禦模式
try:
    result = a / b
except ZeroDivisionError:
    result = default_value
```

```go
// Go -- 類似的防禦模式
if b != 0 {
    quot = a / b
} else {
    quot = 5.0
}
```

在硬體設計中，防禦式程式設計特別重要，因為：

1. **沒有 exception 機制**：硬體沒有 try-catch，必須在邏輯中預防所有邊界情況。
2. **未定義行為更危險**：軟體 crash 可以重啟，硬體產生的錯誤信號可能傳播到整個系統。
3. **Debug 困難**：硬體模擬中追蹤 `NaN` 的傳播非常痛苦，一個除零會污染所有下游計算。

### 為什麼預設值是 5.0？

原始程式碼選擇 `5.0` 作為除零時的預設值。這是一個相對任意的選擇。在實際專案中，你可能會：

- 輸出 `0.0`（最保守）
- 輸出上一個有效值（hold last valid）
- 設置一個 error flag 通知下游

## 運算邏輯

以 stage1 的第一組輸出為例：

- 輸入：`sum = 232.80`, `diff = 36.32`
- 輸出：`prod = 232.80 * 36.32 = 8,455.30`, `quot = 232.80 / 36.32 = 6.41`

由於 `diff` 在這組測試資料中會持續遞減，理論上 `a` 和 `b` 的差距最終可能趨近於零（如果模擬跑夠多 cycle），此時除零防護就會啟動。
