# sc_ptr_flag - 指標旗標

## 概述

`sc_ptr_flag` 是一個模板工具類別，利用指標對齊（alignment）的特性，在指標值的最低有效位元（LSB）中儲存一個布林旗標。這是一種節省記憶體的技巧，不需要額外的布林變數。

**來源檔案**：`sysc/utils/sc_ptr_flag.h`（僅標頭檔）

## 生活比喻

想像每棟大樓的地址都是偶數（2、4、6...）。因為最後一位數永遠是偶數，那個位數就「浪費」了。聰明的郵差可以利用這個空閒的位數來標記：「這棟大樓有掛號信要送」。

同樣的道理：由於物件在記憶體中都是對齊的（至少 2 bytes 對齊），指標值的最低一位元永遠是 0。`sc_ptr_flag` 就利用這個空閒的位元來儲存一個布林旗標。

## 原理

```
假設 T 的對齊要求是 4 bytes：
正常指標值：  ...1100 0100 0000  (最後 2 位元永遠是 00)
可用的空間：  ─────────────┐
                          └── 這個位元可以借用來存旗標！

存了旗標後：  ...1100 0100 0001  (最低位元 = 旗標 true)
取出指標時：  AND 11...1110      (遮掉最低位元即可還原)
```

## 類別介面

```cpp
template<typename T>
class sc_ptr_flag {
public:
    typedef T* pointer;
    typedef T& reference;

    sc_ptr_flag();                          // 預設建構
    sc_ptr_flag(pointer p, bool f = false); // 指標 + 旗標
    sc_ptr_flag& operator=(pointer p);      // 設定指標

    operator pointer() const;     // 隱式轉換為指標
    pointer operator->() const;   // 箭頭運算子
    reference operator*() const;  // 解參照運算子

    pointer get() const;          // 取得指標（不含旗標）
    void reset(pointer p);        // 重設指標（保留旗標）
    bool get_flag() const;        // 取得旗標
    void set_flag(bool f);        // 設定旗標

private:
    uintptr_type m_data;          // 實際儲存：指標 + 旗標
};
```

## 實作細節

### 靜態斷言

```cpp
static_assert(alignof(T) > 1,
    "Unsupported platform/type, need spare LSB of pointer to store flag");
```

確保型別 `T` 的對齊要求至少為 2，否則最低位元不是空閒的，無法使用這個技巧。

### 位元遮罩

```cpp
static const uintptr_type flag_mask    = 0x1;          // 最低位元
static const uintptr_type pointer_mask = ~flag_mask;   // 其餘所有位元
```

### 核心操作

```cpp
// 取得指標：遮掉最低位元
pointer get() const {
    return reinterpret_cast<pointer>(m_data & pointer_mask);
}

// 設定指標：保留旗標
void reset(pointer p) {
    m_data = reinterpret_cast<uintptr_type>(p)
           | static_cast<uintptr_type>(get_flag());
}

// 取得旗標：取最低位元
bool get_flag() const {
    return (m_data & flag_mask) != 0x0;
}

// 設定旗標：只改最低位元
void set_flag(bool f) {
    m_data = (m_data & pointer_mask)
           | static_cast<uintptr_type>(f);
}
```

## 使用場景

在 SystemC 核心中，`sc_ptr_flag` 用於需要同時儲存指標和一個布林屬性的地方，例如：
- 標記某個物件是否已經被處理過
- 標記某個連結是否為特定狀態

這樣做的好處是省掉了一個 `bool` 成員變數的空間（考慮到對齊，一個 `bool` 可能佔用 8 bytes）。

## 記憶體節省示範

```
不使用 sc_ptr_flag：
struct Node {
    Object* ptr;    // 8 bytes
    bool    flag;   // 1 byte + 7 bytes padding = 8 bytes
};                  // 總共 16 bytes

使用 sc_ptr_flag：
struct Node {
    sc_ptr_flag<Object> pf;  // 8 bytes (指標 + 旗標合一)
};                           // 總共 8 bytes
```

## 相關檔案

- [sc_machine.md](sc_machine.md) — 平台偵測（對齊要求依平台而異）
