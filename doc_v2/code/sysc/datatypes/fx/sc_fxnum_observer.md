# sc_fxnum_observer.h / .cpp -- 定點數觀察者

## 概述

`sc_fxnum_observer` 和 `sc_fxnum_fast_observer` 是觀察者模式 (Observer Pattern) 的抽象基底類別，讓使用者可以在定點數物件被建構、解構、讀取或寫入時收到通知。

## 日常類比

就像銀行帳戶的「交易通知」。每次帳戶有存款或提款（寫入），或有人查詢餘額（讀取），銀行都會發一則通知給你。`sc_fxnum_observer` 就是這個通知系統。

## 類別詳情

### `sc_fxnum_observer` -- 任意精度版

```cpp
class sc_fxnum_observer {
protected:
    sc_fxnum_observer() {}
    virtual ~sc_fxnum_observer() {}

public:
    virtual void construct( const sc_fxnum& );  // called on creation
    virtual void destruct( const sc_fxnum& );   // called on destruction
    virtual void read( const sc_fxnum& );       // called on read access
    virtual void write( const sc_fxnum& );      // called on write access

    static sc_fxnum_observer* (*default_observer)();  // factory function pointer
};
```

### `sc_fxnum_fast_observer` -- 有限精度版

結構與 `sc_fxnum_observer` 完全相同，但操作對象是 `sc_fxnum_fast`。

### 預設觀察者

`default_observer` 是一個靜態函式指標，預設為 `NULL`（無觀察者）。使用者可以設定一個工廠函式來自動為所有新建立的定點數物件安裝觀察者。

## 條件編譯

觀察者功能受 `SC_ENABLE_OBSERVERS` 巨集控制：

- **啟用時**：觀察者巨集展開為實際的通知呼叫
- **停用時**：觀察者巨集展開為空操作（零開銷）

```cpp
#ifdef SC_ENABLE_OBSERVERS
#define SC_FXNUM_OBSERVER_READ_(object) \
    SC_OBSERVER_(object, sc_fxnum_observer*, read)
#else
#define SC_FXNUM_OBSERVER_READ_(object)  // empty
#endif
```

## 觀察者巨集

| 巨集 | 觸發時機 |
|------|----------|
| `SC_FXNUM_OBSERVER_CONSTRUCT_` | 定點數物件建構完成 |
| `SC_FXNUM_OBSERVER_DESTRUCT_` | 定點數物件即將解構 |
| `SC_FXNUM_OBSERVER_READ_` | 讀取定點數值 |
| `SC_FXNUM_OBSERVER_WRITE_` | 寫入定點數值 |

快速版有對應的 `SC_FXNUM_FAST_OBSERVER_*` 巨集。

## .cpp 檔案

`sc_fxnum_observer.cpp` 只做一件事：將兩個靜態函式指標初始化為 `0`：

```cpp
sc_fxnum_observer* (*sc_fxnum_observer::default_observer)() = 0;
sc_fxnum_fast_observer* (*sc_fxnum_fast_observer::default_observer)() = 0;
```

## 使用場景

1. **偵錯**：追蹤哪些定點數變數何時被讀寫
2. **統計**：收集定點數運算的統計資訊（多少次量化、多少次溢位）
3. **驗證**：確認特定的值不會被意外修改

## 相關檔案

- `sc_fxdefs.h` -- 通用觀察者巨集 `SC_OBSERVER_`、`SC_OBSERVER_DEFAULT_`
- `sc_fxnum.h` -- 使用觀察者巨集
- `sc_fxval_observer.h` -- 類似的觀察者，用於 `sc_fxval`
