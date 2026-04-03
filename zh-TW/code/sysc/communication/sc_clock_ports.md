# sc_clock_ports -- 時脈專用埠的型別別名

## 概述

`sc_clock_ports.h` 定義了三個型別別名，為時脈訊號提供語意更明確的埠名稱。這些別名純粹是為了向後相容和程式碼可讀性而存在，底層完全就是 `bool` 型別的訊號埠。

**原始檔案：** `sc_clock_ports.h`（僅標頭檔）

## 日常比喻

就像在日常生活中，「鬧鐘」和「時鐘」其實都是「計時器」，只是用途不同所以有不同的名字。`sc_in_clk` 和 `sc_in<bool>` 是完全相同的東西，只是名字告訴讀者「這裡接的是時脈訊號」。

## 定義

```cpp
typedef sc_in<bool>    sc_in_clk;
typedef sc_inout<bool> sc_inout_clk;
typedef sc_out<bool>   sc_out_clk;
```

就是這麼簡單 -- 三行 `typedef`。

## 對應關係

| 時脈別名 | 等同於 | 用途 |
|----------|--------|------|
| `sc_in_clk` | `sc_in<bool>` | 接收時脈輸入（最常用） |
| `sc_inout_clk` | `sc_inout<bool>` | 雙向時脈（極少見） |
| `sc_out_clk` | `sc_out<bool>` | 時脈輸出（極少見） |

## 使用範例概念

```cpp
SC_MODULE(FlipFlop) {
    sc_in_clk clk;          // same as sc_in<bool> clk;
    sc_in<bool> d;
    sc_out<bool> q;

    void process() {
        q.write(d.read());
    }

    SC_CTOR(FlipFlop) {
        SC_METHOD(process);
        sensitive << clk.pos();  // trigger on clock positive edge
    }
};
```

## 設計重點

### 為什麼不用模板特化？

這些型別別名是在 SystemC 2.0 時代引入的，當時的編碼慣例傾向使用 `typedef`。它們作為向後相容的機制保留下來，讓舊的程式碼不需要修改。

### 何時使用 `sc_in_clk` vs `sc_in<bool>`？

兩者功能完全相同。建議：
- 如果埠確實是用來接收時脈訊號，使用 `sc_in_clk` 提高可讀性
- 如果是一般的布林訊號（如 reset、enable），使用 `sc_in<bool>`

## 相關檔案

- `sc_signal_ports.h` - `sc_in<bool>` 等類別的定義
- `sc_clock.h` - 時脈通道，通常連接到 `sc_in_clk`
