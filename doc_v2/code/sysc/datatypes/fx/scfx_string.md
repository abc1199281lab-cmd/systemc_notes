# scfx_string.h -- 內部字串類別

## 概述

`scfx_string` 是一個**簡單的動態字串類別**，專為定點數的字串轉換而設計。它不使用 `std::string`，而是自行管理記憶體緩衝區，提供逐字元附加和索引操作。

## 日常類比

`scfx_string` 就像一張「可撕式便條紙」。你可以一個字一個字地寫上去，寫滿了自動換一張更大的紙。你也可以撕掉（discard）尾巴的幾個字，或者從中間移除（remove）一個字。

## 類別詳情

### 成員變數

| 成員 | 型別 | 說明 |
|------|------|------|
| `m_len` | `size_t` | 目前字串長度 |
| `m_alloc` | `size_t` | 已分配的緩衝區大小 |
| `m_buffer` | `char*` | 字元緩衝區 |

### 主要方法

| 方法 | 說明 |
|------|------|
| `scfx_string()` | 建構，初始大小 = `BUFSIZ` |
| `length()` | 取得目前長度 |
| `clear()` | 清空（長度歸零，不釋放記憶體） |
| `operator[](i)` | 存取第 i 個字元（自動擴展） |
| `operator+=(char)` | 附加一個字元 |
| `operator+=(const char*)` | 附加一個 C 字串 |
| `append(n)` | 將長度增加 n（假設已透過 [] 填入） |
| `discard(n)` | 將長度減少 n（截斷尾部） |
| `remove(i)` | 移除第 i 個字元，後面的字元前移 |
| `operator const char*()` | 隱式轉換為 C 字串 |

### 自動擴展

當透過 `operator[]` 存取超出目前分配大小的位置時，會自動呼叫 `resize()`，將緩衝區大小加倍直到足夠：

```cpp
void resize(size_t i) {
    do {
        m_alloc *= 2;
    } while (i >= m_alloc);
    // allocate new buffer, copy old content
}
```

## 為什麼不用 std::string？

1. **效能**：定點數的字串轉換是高頻操作，自訂的簡單實作避免了 `std::string` 的引用計數和小字串最佳化的開銷
2. **歷史原因**：`scfx_string` 早於 C++ 標準化 `std::string` 的時期
3. **介面需求**：`append(n)` 和 `discard(n)` 等操作在 `std::string` 中沒有直接對應

## 相關檔案

- `scfx_utils.h` -- 使用 `scfx_string` 進行格式化輸出
- `scfx_rep.h` / `scfx_rep.cpp` -- `to_string()` 方法使用 `scfx_string`
