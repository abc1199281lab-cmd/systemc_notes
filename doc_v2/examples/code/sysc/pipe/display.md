# display -- 結果顯示模組

> **檔案**: `display.h`, `display.cpp` | **角色**: 資料接收端（data sink）

## 軟體類比

`display` 是管線的終點，負責把最終結果印出來。它就像：

- Unix pipe 中最後一個指令的 stdout
- Log sink / console appender
- ETL 管線的 Load 步驟（把結果寫到某處）

```python
# Python 類比
def display(value):
    print(f"result = {value}")
```

## 模組結構

### Header（display.h）

```cpp
SC_MODULE(display) {
    sc_in<bool>   clk;
    sc_in<double> in;     // 來自 stage3.powr

    void entry();

    SC_CTOR(display) {
        SC_METHOD(entry);
        dont_initialize();
        sensitive << clk.pos();
    }
};
```

### Implementation（display.cpp）

```cpp
void display::entry() {
    printf("result = %g\n", in.read());
}
```

## 關鍵概念

### 最簡單的模組

`display` 是整個管線中最簡單的模組。它只有一個輸入 port，沒有輸出 port，只做一件事：讀取輸入並印出。

這個模組展示了 SystemC 的一個重要設計哲學：**即使是最簡單的功能也包裝成獨立模組**。在軟體工程中，這等同於 single responsibility principle -- 每個模組只負責一件事。

### printf vs cout

這個範例使用 C 風格的 `printf` 而不是 C++ 的 `cout`。在 SystemC 中兩者都可以用，但 `printf` 在某些情況下有優勢：

- 格式控制更直觀（`%g` 自動選擇最佳的浮點數顯示格式）
- 不需要 include `<iostream>`
- 在多 process 環境下，`printf` 通常是 atomic 的（一次印完一行）

`%g` 格式說明符會自動在 `%f`（定點）和 `%e`（科學記號）之間選擇較短的格式，適合顯示範圍不確定的浮點數。

### Sink 模組的角色

在硬體設計中，sink 模組通常不會只是 `printf`。在真實系統中，它可能是：

- **DAC（數位類比轉換器）**：把數位信號轉成電壓輸出
- **UART TX**：把資料透過串列埠送出
- **Memory controller**：把結果寫入記憶體

在 SystemC 模擬中，`printf` 是最常見的 debug 和驗證手段，相當於軟體中的 `console.log`。
