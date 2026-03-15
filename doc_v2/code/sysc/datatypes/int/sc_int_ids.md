# sc_int_ids.h — 整數子系統錯誤報告識別碼

## 概述

`sc_int_ids.h` 定義了整數子系統（`datatypes/int/`）使用的錯誤報告識別碼。當整數運算發生錯誤時（如初始化失敗、賦值溢位、運算錯誤、型別轉換失敗），SystemC 的報告機制會使用這些識別碼來產生有意義的錯誤訊息。

**源檔案：**
- `ref/systemc/src/sysc/datatypes/int/sc_int_ids.h`

## 日常類比

這就像工廠裡的「錯誤代碼表」。每台機器出問題時會顯示一個代碼（如 E400），工人查表就知道發生了什麼事，而不是只看到「機器壞了」這種模糊的訊息。

## 定義的識別碼

| 識別碼 | 編號 | 說明 |
|--------|------|------|
| `SC_ID_INIT_FAILED_` | 400 | 初始化失敗 |
| `SC_ID_ASSIGNMENT_FAILED_` | 401 | 賦值失敗 |
| `SC_ID_OPERATION_FAILED_` | 402 | 運算失敗 |
| `SC_ID_CONVERSION_FAILED_` | 403 | 型別轉換失敗 |

這些識別碼的編號範圍是 400-499，專門保留給 `datatypes/int` 子系統使用。

## 實作機制

使用 `SC_DEFINE_MESSAGE` 巨集來宣告：

```cpp
SC_DEFINE_MESSAGE( SC_ID_INIT_FAILED_, 400, "initialization failed" )
```

這個巨集在不同的編譯環境下有不同的展開方式，但核心功能是將識別碼字串與錯誤訊息關聯起來，讓 `sc_report` 機制可以使用。

## 相關檔案

- [sc_int_base.md](sc_int_base.md) — 使用這些識別碼報告錯誤
- [sc_uint_base.md](sc_uint_base.md) — 使用這些識別碼報告錯誤
- [sc_nbutils.md](sc_nbutils.md) — 使用這些識別碼報告錯誤
