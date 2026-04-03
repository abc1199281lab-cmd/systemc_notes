# sc_fxval_observer.h / .cpp -- 定點數值觀察者

## 概述

`sc_fxval_observer` 和 `sc_fxval_fast_observer` 提供觀察者模式的抽象基底類別，用於監控 `sc_fxval` 和 `sc_fxval_fast` 物件的建構、解構、讀取和寫入事件。

## 日常類比

就像快遞的「物流追蹤」系統。每當包裹（值）被建立、移動或拆封時，系統都會記錄一筆追蹤紀錄。

## 與 `sc_fxnum_observer` 的差異

| 特性 | `sc_fxval_observer` | `sc_fxnum_observer` |
|------|---------------------|---------------------|
| 監控對象 | `sc_fxval`（中間值） | `sc_fxnum`（定點數變數） |
| 使用場景 | 追蹤運算過程 | 追蹤變數狀態 |
| 啟用方式 | `SC_ENABLE_OBSERVERS` | `SC_ENABLE_OBSERVERS` |

## 類別詳情

### `sc_fxval_observer`

```cpp
class sc_fxval_observer {
protected:
    sc_fxval_observer() {}
    virtual ~sc_fxval_observer() {}

public:
    virtual void construct( const sc_fxval& );
    virtual void destruct( const sc_fxval& );
    virtual void read( const sc_fxval& );
    virtual void write( const sc_fxval& );

    static sc_fxval_observer* (*default_observer)();
};
```

所有虛擬方法的預設實作都是空的（no-op），使用者需要繼承並覆寫。

### 條件編譯巨集

```cpp
#ifdef SC_ENABLE_OBSERVERS
#define SC_FXVAL_OBSERVER_READ_(object) \
    SC_OBSERVER_(object, sc_fxval_observer*, read)
#else
#define SC_FXVAL_OBSERVER_READ_(object)  // empty
#endif
```

## .cpp 檔案

`sc_fxval_observer.cpp` 將靜態工廠函式指標初始化為 `0`：

```cpp
sc_fxval_observer* (*sc_fxval_observer::default_observer)() = 0;
sc_fxval_fast_observer* (*sc_fxval_fast_observer::default_observer)() = 0;
```

## 相關檔案

- `sc_fxval.h` -- 使用觀察者巨集的值類別
- `sc_fxnum_observer.h` -- 類似的觀察者，用於 `sc_fxnum`
- `sc_fxdefs.h` -- 通用觀察者巨集定義
