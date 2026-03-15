# sc_bit - 單一位元類別（已棄用）

## 概述

`sc_bit` 是 SystemC 中最基本的位元型別，表示一個只有 0 或 1 的值。**此類別已被棄用**，建議直接使用 C++ 原生的 `bool` 型別。建構時會觸發棄用警告。

**原始檔案：** `sc_bit.h` + `sc_bit.cpp`

## 日常比喻

`sc_bit` 就像一個最簡單的電燈開關——只有「開」和「關」兩種狀態。但現在 C++ 自己就有 `bool`（也是開/關），所以不需要再額外包裝一個開關了。就像你不需要買一個專門的「開關轉接器」來控制一個已經有開關的燈。

## 關鍵概念

### 為什麼被棄用？

早期 SystemC 設計時，C++ 的 `bool` 型別還不夠標準化，所以需要 `sc_bit` 來確保跨平台一致性。現在 C++ 標準已經完善了 `bool`，`sc_bit` 就變得多餘了。

### 棄用機制

每次建構 `sc_bit` 時，會呼叫 `sc_deprecated_sc_bit()`，此函式使用靜態變數確保警告只印出一次：

```cpp
sc_bit my_bit(1);
// First time: prints "sc_bit is deprecated, use bool instead"
// Subsequent times: no warning
```

## 類別介面

### 建構子

```cpp
sc_bit();                // default: false
sc_bit(bool a);          // from bool
sc_bit(char a);          // '0' or '1'
sc_bit(int a);           // 0 or 1
sc_bit(const sc_logic& a); // from sc_logic (must be 0 or 1)
```

所有建構子都是 `explicit`，避免隱式轉換帶來的意外。

### 值轉換

```cpp
bool to_bool() const;    // convert to bool
char to_char() const;    // convert to '0' or '1'
```

### 位元運算

```cpp
sc_bit& operator &= (const sc_bit& b);  // AND
sc_bit& operator |= (const sc_bit& b);  // OR
sc_bit& operator ^= (const sc_bit& b);  // XOR
const sc_bit operator ~ () const;        // NOT
```

位元運算在內部其實就是布林運算：AND 對應 `&&`，OR 對應 `||`，XOR 對應 `!=`。

### I/O

```cpp
void print(std::ostream& os) const;
void scan(std::istream& is);
friend std::ostream& operator<<(std::ostream&, const sc_bit&);
friend std::istream& operator>>(std::istream&, sc_bit&);
```

## 內部實作

- 內部儲存為一個 `bool m_val`
- `to_value()` 系列靜態方法負責驗證輸入值的合法性
- 非法值（如 `char` 不是 '0' 或 '1'）會呼叫 `invalid_value()` 觸發 `SC_REPORT_ERROR` 並呼叫 `sc_abort()`

## 錯誤處理

| 情境 | 錯誤 |
|------|------|
| `sc_bit('2')` | `SC_ID_VALUE_NOT_VALID_` |
| `sc_bit(5)` | `SC_ID_VALUE_NOT_VALID_` |

## 設計理由 / RTL 背景

在 RTL（暫存器傳輸層）設計中，單一位元是最基本的訊號單位。Verilog 的 `reg` 和 VHDL 的 `std_logic` 都有對應的單位元型別。`sc_bit` 最初是為了 VSIA（Virtual Socket Interface Alliance）相容性而設計的，但隨著 C++ 標準的成熟，`bool` 已足以勝任。

## 相關檔案

- [sc_logic.md](sc_logic.md) - 四值邏輯型別，`sc_bit` 的進階版本
- [sc_bit_ids.md](sc_bit_ids.md) - 錯誤訊息 ID 定義
- 原始碼：`ref/systemc/src/sysc/datatypes/bit/sc_bit.h`
- 原始碼：`ref/systemc/src/sysc/datatypes/bit/sc_bit.cpp`
