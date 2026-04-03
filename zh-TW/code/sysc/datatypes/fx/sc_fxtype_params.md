# sc_fxtype_params.h / .cpp -- 定點數型別參數

## 概述

`sc_fxtype_params` 封裝了定點數型別的五個核心參數：位寬 (wl)、整數位寬 (iwl)、量化模式 (q_mode)、溢位模式 (o_mode) 和飽和位數 (n_bits)。它是定點數「格式描述」的核心資料結構。

## 日常類比

如果定點數是一張照片，`sc_fxtype_params` 就是相機的設定：
- **位寬 (wl)** = 解析度（總共多少像素）
- **整數位寬 (iwl)** = 焦距（決定能拍多遠的景物）
- **量化模式 (q_mode)** = 壓縮演算法（JPEG 品質設定）
- **溢位模式 (o_mode)** = 過曝處理（飽和到白色 vs. 自動調整）
- **飽和位數 (n_bits)** = 特殊曝光參數

## 類別詳情

### 成員變數

| 成員 | 型別 | 說明 |
|------|------|------|
| `m_wl` | `int` | 總位寬 (Word Length)，必須 > 0 |
| `m_iwl` | `int` | 整數位寬 (Integer Word Length)，可以是任意整數 |
| `m_q_mode` | `sc_q_mode` | 量化模式 |
| `m_o_mode` | `sc_o_mode` | 溢位模式 |
| `m_n_bits` | `int` | 飽和位數，必須 >= 0 |

**注意：** `iwl` 可以大於 `wl`（表示沒有小數位）或為負數（表示所有位都在小數點後面）。小數位寬 = `wl - iwl`。

### 建構函式

| 建構方式 | 行為 |
|----------|------|
| `sc_fxtype_params()` | 從 `sc_fxtype_context` 取得預設值 |
| `sc_fxtype_params(wl, iwl)` | 指定位寬，其餘取預設值 |
| `sc_fxtype_params(q, o, n)` | 指定模式，位寬取預設值 |
| `sc_fxtype_params(wl, iwl, q, o, n)` | 完全指定所有參數 |
| `sc_fxtype_params(params, wl, iwl)` | 複製並覆蓋位寬 |
| `sc_fxtype_params(params, q, o, n)` | 複製並覆蓋模式 |
| `sc_fxtype_params(sc_without_context)` | 使用編譯時預設值 |

### 位寬的含義

```
    整數部分 (iwl 位)     小數部分 (wl - iwl 位)
    ←──────────────→     ←──────────────────────→
    [sign] [integer bits] . [fractional bits]
    ←────────────────── wl 位 ──────────────────→
```

範例：`sc_fixed<8, 4>` 表示：
- wl = 8（共 8 位元）
- iwl = 4（4 位整數，含符號位）
- 小數部分 = 4 位
- 可表示範圍：-8.0 到 +7.9375（步進 0.0625 = 2^-4）

### `sc_fxtype_context`

```cpp
typedef sc_context<sc_fxtype_params> sc_fxtype_context;
```

使用範例：

```cpp
{
    sc_fxtype_context ctx(sc_fxtype_params(16, 8, SC_RND, SC_SAT));
    // All sc_fix/sc_ufix created here default to 16-bit, 8-integer-bit
    sc_fix a;  // wl=16, iwl=8, SC_RND, SC_SAT
    sc_fix b;  // same defaults
}
```

## .cpp 檔案

`sc_fxtype_params.cpp` 包含：

1. 模板實例化 `sc_global<sc_fxtype_params>` 和 `sc_context<sc_fxtype_params>`
2. `to_string()` -- 輸出格式如 `"(16,8,SC_RND,SC_SAT,0)"`
3. `print()` -- 同 `to_string()`
4. `dump()` -- 詳細輸出每個欄位

## 相關檔案

- `sc_fxdefs.h` -- 列舉和預設值定義
- `sc_context.h` -- `sc_context<T>` 模板
- `scfx_params.h` -- 在 `scfx_params` 中組合使用
- `sc_fxnum.h` -- 使用 `sc_fxtype_params` 來參數化 `sc_fxnum`
