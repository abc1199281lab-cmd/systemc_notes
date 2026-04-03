# sc_fx_ids.h -- 定點數錯誤訊息 ID

## 概述

`sc_fx_ids.h` 定義了定點數子系統所有的**錯誤報告 ID**。這些 ID 與 SystemC 的錯誤報告機制 (`sc_report`) 配合使用，讓使用者在偵錯時能看到清楚的錯誤訊息。

## 日常類比

就像汽車儀表板上的錯誤碼一樣 -- 引擎故障是 P0300、油壓異常是 P0520。每個錯誤碼對應一個明確的問題描述，讓維修技師能快速定位問題。

## 錯誤 ID 列表

所有 ID 的編號範圍為 **300-399**（分配給 datatypes/fx 模組）。

| ID | 編號 | 錯誤訊息 | 觸發條件 |
|----|------|----------|----------|
| `SC_ID_INVALID_WL_` | 300 | "total wordlength <= 0 is not valid" | 總位寬設為 0 或負數 |
| `SC_ID_INVALID_N_BITS_` | 301 | "number of bits < 0 is not valid" | 飽和位數為負數 |
| `SC_ID_INVALID_DIV_WL_` | 302 | "division wordlength <= 0 is not valid" | 除法位寬設為 0 或負數 |
| `SC_ID_INVALID_CTE_WL_` | 303 | "constant wordlength <= 0 is not valid" | 常數位寬設為 0 或負數 |
| `SC_ID_INVALID_MAX_WL_` | 304 | "maximum wordlength <= 0 and != -1 is not valid" | 最大位寬非法 |
| `SC_ID_INVALID_FX_VALUE_` | 305 | "invalid fixed-point value" | 值為 NaN 或 Inf |
| `SC_ID_INVALID_O_MODE_` | 306 | "invalid overflow mode" | 溢位模式不合法 |
| `SC_ID_OUT_OF_RANGE_` | 307 | "index out of range" | 位元索引超出範圍 |
| `SC_ID_CONTEXT_BEGIN_FAILED_` | 308 | "context begin failed" | 重複呼叫 context begin |
| `SC_ID_CONTEXT_END_FAILED_` | 309 | "context end failed" | 在未啟動的 context 上呼叫 end |
| `SC_ID_WRAP_SM_NOT_DEFINED_` | 310 | "SC_WRAP_SM not defined for unsigned numbers" | 對無號數使用 sign magnitude wrap |

## 實作機制

使用 `SC_DEFINE_MESSAGE` 巨集定義每個 ID：

```cpp
SC_DEFINE_MESSAGE( SC_ID_INVALID_WL_, 300,
    "total wordlength <= 0 is not valid" )
```

這個巨集會在 `sc_core` 命名空間中宣告一個 `extern const char[]`，實際的字串定義在 `sc_report_handler.cpp` 中。

## 使用方式

這些 ID 透過 `sc_fxdefs.h` 中的檢查巨集被使用：

```cpp
// In sc_fxdefs.h:
#define SC_CHECK_WL_(wl) \
    SC_ERROR_IF_( (wl) <= 0, sc_core::SC_ID_INVALID_WL_ )

// Usage in code:
sc_fxtype_params::wl( int wl_ ) {
    SC_CHECK_WL_( wl_ );  // triggers error if wl_ <= 0
    m_wl = wl_;
}
```

## 相關檔案

- `sc_fxdefs.h` -- 使用這些 ID 的檢查巨集
- `sysc/utils/sc_report.h` -- 錯誤報告基礎設施
- `sc_context.h` -- 使用 `SC_ID_CONTEXT_BEGIN_FAILED_` / `SC_ID_CONTEXT_END_FAILED_`
- `scfx_params.h` -- 使用 `SC_ID_INVALID_O_MODE_`
