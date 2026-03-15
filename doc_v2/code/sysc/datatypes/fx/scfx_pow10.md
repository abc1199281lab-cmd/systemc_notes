# scfx_pow10.h / .cpp -- 10 的次方計算與快取

## 概述

`scfx_pow10` 類別用於計算和快取 **10 的次方** 在任意精度表示下的值。這主要用於字串與定點數之間的十進位轉換（例如 `"3.14"` -> 內部表示）。

## 日常類比

想像你在做十進位到二進位的轉換，需要反覆用到 10、100、1000、0.1、0.01 等數值。每次重新計算很浪費時間，所以你把這些值寫在一張「查表」上。`scfx_pow10` 就是這張查表，而且它會在第一次需要時才計算，之後就直接查表。

## 類別詳情

```cpp
class scfx_pow10 {
public:
    scfx_pow10();
    ~scfx_pow10();

    scfx_rep operator() (int);  // returns 10^n as scfx_rep

private:
    scfx_rep* pos(int);  // compute positive power
    scfx_rep* neg(int);  // compute negative power

    scfx_rep m_pos[SCFX_POW10_TABLE_SIZE]; // 10^0, 10^1, ..., 10^31
    scfx_rep m_neg[SCFX_POW10_TABLE_SIZE]; // 10^0, 10^-1, ..., 10^-31
};
```

### 快取表

| 陣列 | 大小 | 內容 |
|------|------|------|
| `m_pos[32]` | 32 | 10^0 到 10^31 的任意精度值 |
| `m_neg[32]` | 32 | 10^0 到 10^-31 的任意精度值 |

### `operator()` 用法

```cpp
scfx_pow10 pow10;
scfx_rep val = pow10(5);   // returns 10^5 = 100000
scfx_rep val2 = pow10(-3); // returns 10^-3 = 0.001
```

對於超出快取範圍的指數（|n| > 31），會通過已快取的值進行組合計算。

## 為什麼需要任意精度的 10 的次方？

浮點數無法精確表示 0.1、0.01 等十進位小數（因為它們是二進位的無窮小數）。在定點數的字串轉換中，需要精確的十進位計算，所以必須使用 `scfx_rep` 來表示 10 的次方。

## 相關檔案

- `scfx_rep.h` -- `scfx_rep` 類別，用於儲存任意精度值
- `scfx_rep.cpp` -- 字串解析中使用 `scfx_pow10`
