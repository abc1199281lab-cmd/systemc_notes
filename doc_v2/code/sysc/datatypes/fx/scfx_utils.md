# scfx_utils.h / .cpp -- 定點數工具函式

## 概述

`scfx_utils.h` 提供了定點數字串解析和格式化所需的各種**工具函式**。這些函式處理數值的進制前綴解析、位元搜尋、數字字元判斷等底層操作。

## 日常類比

這個檔案就像一個「工具箱」，裝著各種螺絲起子、扳手和測量工具。單獨來看每個工具都很簡單，但它們是組裝大型系統（字串轉換、數值顯示）不可或缺的零件。

## 位元搜尋函式

### `scfx_find_msb()` -- 找最高有效位

```cpp
int scfx_find_msb(unsigned long x);
```

使用二分搜尋法找到 `x` 中最高的非零位元位置。例如 `scfx_find_msb(0b10100)` 返回 4。

### `scfx_find_lsb()` -- 找最低有效位

```cpp
int scfx_find_lsb(unsigned long x);
```

找到 `x` 中最低的非零位元位置。例如 `scfx_find_lsb(0b10100)` 返回 2。

## 字串解析函式

### `scfx_parse_sign()` -- 解析符號

```cpp
int scfx_parse_sign(const char*& s, bool& sign_char);
```

從字串開頭讀取 `+` 或 `-`，返回 1 或 -1，並推進指標。

### `scfx_parse_prefix()` -- 解析進制前綴

```cpp
sc_numrep scfx_parse_prefix(const char*& s);
```

解析字串前綴來判斷進制：

| 前綴 | 進制 | 返回值 |
|------|------|--------|
| `0b` | 二進位 | `SC_BIN` |
| `0bus` | 二進位無號 | `SC_BIN_US` |
| `0bsm` | 二進位符號-大小 | `SC_BIN_SM` |
| `0o` | 八進位 | `SC_OCT` |
| `0x` | 十六進位 | `SC_HEX` |
| `0d` | 十進位 | `SC_DEC` |
| `0csd` | CSD (Canonical Signed Digit) | `SC_CSD` |
| 無前綴 | 十進位 | `SC_DEC` |

### `scfx_parse_base()` -- 解析基數

```cpp
int scfx_parse_base(const char*& s);
```

類似 `scfx_parse_prefix()`，但返回數值基數（2, 8, 10, 16）。

### 字元判斷函式

```cpp
bool scfx_is_digit(char c, sc_numrep numrep);  // is valid digit?
int scfx_to_digit(char c, sc_numrep numrep);   // char to numeric value
bool scfx_is_nan(const char* s);                // is "NaN"?
bool scfx_is_inf(const char* s);                // is "Inf" or "Infinity"?
bool scfx_exp_start(const char* s);             // starts with "e+" or "e-"?
```

## 字串輸出函式

### `scfx_print_nan()` / `scfx_print_inf()`

```cpp
void scfx_print_nan(scfx_string& s);
void scfx_print_inf(scfx_string& s, bool negative);
```

### `scfx_print_prefix()` -- 印出進制前綴

```cpp
void scfx_print_prefix(scfx_string& s, sc_numrep numrep);
```

### `scfx_print_exp()` -- 印出指數

```cpp
void scfx_print_exp(scfx_string& s, int exp);
```

格式如 `"e+10"` 或 `"e-3"`，跳過前導零。

## CSD 轉換

```cpp
void scfx_tc2csd(scfx_string&, int);  // two's complement to CSD
void scfx_csd2tc(scfx_string&);       // CSD to two's complement
```

CSD (Canonical Signed Digit) 是一種數位表示法，每個位元可以是 0、1 或 -1，保證相鄰位元不會同時非零。這在硬體乘法器設計中用於減少加法器數量。

## .cpp 檔案

`scfx_utils.cpp` 主要實作 CSD 轉換的 `scfx_tc2csd()` 和 `scfx_csd2tc()` 函式。

## 相關檔案

- `scfx_string.h` -- `scfx_string` 類別
- `scfx_rep.h` / `scfx_rep.cpp` -- 使用這些工具進行字串轉換
- `sc_fxdefs.h` -- `sc_numrep` 列舉定義
- `scfx_params.h` -- 被本檔案 include
