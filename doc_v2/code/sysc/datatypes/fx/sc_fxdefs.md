# sc_fxdefs.h / sc_fxdefs.cpp -- 定點數定義與列舉

## 概述

`sc_fxdefs.h` 定義了定點數系統的所有**列舉型別 (enumerations)**、**預設常數**和**檢查巨集**。它是整個定點數子系統的「規格書」，決定了各種行為模式的名稱和預設值。

## 日常類比

這個檔案就像一個「選單」或「設定面板」。就像手機的螢幕設定有「亮度」、「色溫」、「字體大小」等選項，`sc_fxdefs.h` 列出了定點數所有可調整的「設定項目」。

## 列舉型別

### `sc_enc` -- 編碼方式

決定數值的符號表示法：

| 值 | 含義 | 說明 |
|----|------|------|
| `SC_TC_` | Two's Complement | 二補數（有號），就像日常的正負數 |
| `SC_US_` | Unsigned | 無號數，只有正數和零 |

### `sc_q_mode` -- 量化模式

當小數部分超出可表示範圍時，怎麼處理多餘的位元：

| 值 | 含義 | 類比 |
|----|------|------|
| `SC_RND` | Round to plus infinity | 標準四捨五入（偏向正無窮） |
| `SC_RND_ZERO` | Round to zero | 向零四捨五入 |
| `SC_RND_MIN_INF` | Round to minus infinity | 向負無窮四捨五入 |
| `SC_RND_INF` | Round to infinity | 向遠離零的方向四捨五入 |
| `SC_RND_CONV` | Convergent rounding | 銀行家進位法（最接近偶數） |
| `SC_TRN` | Truncation | 直接截斷（向負無窮） |
| `SC_TRN_ZERO` | Truncation to zero | 直接截斷（向零） |

**銀行家進位法的類比：** 想像你每天把零錢丟進存錢筒。如果總是四捨五入，長期來看你會多存一點（因為 0.5 總是進位）。銀行家進位法在 0.5 的時候選擇進位到偶數，長期來看比較公平。

### `sc_o_mode` -- 溢位模式

當數值超出可表示範圍時怎麼辦：

| 值 | 含義 | 類比 |
|----|------|------|
| `SC_SAT` | Saturation | 卡在最大/最小值（像音量旋鈕轉到底） |
| `SC_SAT_ZERO` | Saturation to zero | 直接歸零 |
| `SC_SAT_SYM` | Symmetrical saturation | 對稱飽和（正負範圍相同） |
| `SC_WRAP` | Wrap-around | 繞回（像里程表 999999 -> 000000） |
| `SC_WRAP_SM` | Sign magnitude wrap-around | 符號-大小繞回 |

### `sc_switch` -- 開關

```cpp
enum sc_switch { SC_OFF, SC_ON };
```

用於控制 cast switch 的開啟/關閉。

### `sc_fmt` -- 輸出格式

```cpp
enum sc_fmt { SC_F, SC_E };
```

| 值 | 含義 | 範例 |
|----|------|------|
| `SC_F` | Fixed format | `3.14159` |
| `SC_E` | Scientific format | `3.14159e+0` |

## 預設常數

### 型別參數預設值

| 常數 | 值 | 說明 |
|------|----|------|
| `SC_BUILTIN_WL_` | 32 | 預設總位寬 |
| `SC_BUILTIN_IWL_` | 32 | 預設整數位寬 |
| `SC_BUILTIN_Q_MODE_` | `SC_TRN` | 預設量化模式（截斷） |
| `SC_BUILTIN_O_MODE_` | `SC_WRAP` | 預設溢位模式（繞回） |
| `SC_BUILTIN_N_BITS_` | 0 | 預設飽和位數 |
| `SC_BUILTIN_CAST_SWITCH_` | `SC_ON` | 預設開啟 cast |

### 值型別預設值

| 常數 | 值 | 說明 |
|------|----|------|
| `SC_BUILTIN_DIV_WL_` | 64 | 除法運算的位寬 |
| `SC_BUILTIN_CTE_WL_` | 64 | 常數的位寬 |
| `SC_BUILTIN_MAX_WL_` | 1024 | 最大允許位寬 |

這些值可以透過編譯時的 `SC_FXDIV_WL`、`SC_FXCTE_WL`、`SC_FXMAX_WL` 巨集覆蓋。

## 檢查巨集

```cpp
SC_CHECK_WL_(wl)       // wl > 0
SC_CHECK_N_BITS_(n)    // n >= 0
SC_CHECK_DIV_WL_(wl)   // wl > 0
SC_CHECK_CTE_WL_(wl)   // wl > 0
SC_CHECK_MAX_WL_(wl)   // wl > 0 or wl == -1
```

這些巨集在參數非法時觸發 `SC_REPORT_ERROR` 並中止程式。

## Observer 巨集

```cpp
SC_OBSERVER_(object, observer_type, event)
SC_OBSERVER_DEFAULT_(observer_type)
```

為觀察者模式提供通用巨集，用於在定點數物件被讀寫時通知觀察者。

## sc_fxdefs.cpp

`.cpp` 檔案實作了每個列舉的 `to_string()` 函式，將列舉值轉為可讀字串（例如 `SC_TC_` -> `"SC_TC_"`）。

## 相關檔案

- `sc_fx_ids.h` -- 錯誤訊息 ID（被檢查巨集使用）
- `sc_fxtype_params.h` -- 使用這些列舉和常數
- `sc_fxcast_switch.h` -- 使用 `sc_switch` 列舉
- `sc_fxnum_observer.h` -- 使用 observer 巨集
