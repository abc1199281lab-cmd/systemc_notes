# sc_unsigned — 任意精度無號整數

## 概述

`sc_unsigned` 是 SystemC 中任意精度無號整數的核心類別，也是 `sc_biguint<W>` 的基底類別。它與 `sc_signed` 是姐妹類別，共享大部分實作程式碼，但所有運算都以無號方式進行。

一個重要特性：**n 位元的 `sc_unsigned` 等同於 (n+1) 位元的非負 `sc_signed`**。這是因為 `sc_unsigned` 需要額外一個位元來表示「符號位為 0」（正數）。

**源檔案：**
- `ref/systemc/src/sysc/datatypes/int/sc_unsigned.h`
- `ref/systemc/src/sysc/datatypes/int/sc_unsigned.cpp`
- `ref/systemc/src/sysc/datatypes/int/sc_unsigned_inlines.h`
- `ref/systemc/src/sysc/datatypes/int/sc_unsigned_friends.h`

## 日常類比

`sc_unsigned` 就像一台「超大型里程表」——可以有任意多位數，但永遠不會顯示負數。需要 256 位元的位址空間？沒問題。需要 1024 位元的加密金鑰？也可以。

## 類別結構

```mermaid
classDiagram
    class sc_value_base {
        <<abstract>>
    }

    class sc_unsigned {
        -sc_digit* digit
        -int nbits
        -int ndigits
        -small_type sgn
        +sc_unsigned(int nb)
        +operator=(various)
        +operator[](int i) sc_unsigned_bitref
        +range(int hi, int lo) sc_unsigned_subref
        +to_uint() unsigned int
        +to_uint64() uint64
        +length() int
    }

    sc_value_base <|-- sc_unsigned
    sc_unsigned <|-- sc_biguint_W["sc_biguint&lt;W&gt;"]
```

## 核心概念

### 1. 內部表示法

與 `sc_signed` 完全相同，使用 sign-magnitude 表示法：

```cpp
small_type sgn;      // always SC_POS or SC_ZERO (never SC_NEG)
sc_digit*  digit;    // magnitude array
int        nbits;    // number of bits
int        ndigits;  // number of sc_digit elements
```

因為是無號整數，`sgn` 永遠不會是 `SC_NEG`。

### 2. 與 sc_signed 的程式碼共享

`sc_unsigned` 和 `sc_signed` 共享大部分實作程式碼。原始碼中使用巨集來控制差異：

```cpp
// In shared implementation files:
#ifdef IF_SC_SIGNED
    // signed-specific code
#else
    // unsigned-specific code
#endif
```

### 3. 一元運算子的特殊行為

```cpp
sc_unsigned operator + (const sc_unsigned& u);  // returns sc_unsigned
sc_signed   operator - (const sc_unsigned& u);  // negation returns sc_signed!
sc_signed   operator ~ (const sc_unsigned& u);  // bitwise NOT returns sc_signed!
```

注意：對 `sc_unsigned` 取負值或位元反轉，結果是 `sc_signed`。因為負數無法用無號型別表示。

### 4. 混合運算語意

| 運算 | 結果型別 |
|------|----------|
| `sc_unsigned + sc_unsigned` | `sc_unsigned` |
| `sc_unsigned + sc_signed` | `sc_signed` |
| `sc_unsigned - sc_unsigned` | `sc_signed` |
| `-sc_unsigned` | `sc_signed` |

### 5. 代理類別

與 `sc_signed` 對稱，提供四個代理類別：
- `sc_unsigned_bitref_r` / `sc_unsigned_bitref`：位元選取
- `sc_unsigned_subref_r` / `sc_unsigned_subref`：範圍選取

### 6. 檔案分工

| 檔案 | 職責 |
|------|------|
| `sc_unsigned.h` | 類別宣告與代理類別 |
| `sc_unsigned.cpp` | 核心實作 |
| `sc_unsigned_inlines.h` | 延遲定義的 inline 函式 |
| `sc_unsigned_friends.h` | friend 運算子宣告 |

## 使用範例

```cpp
sc_unsigned addr(256);  // 256-bit address
addr = "0x1234_5678_9ABC_DEF0_1234_5678_9ABC_DEF0";

sc_unsigned key(1024);  // 1024-bit encryption key

// Arithmetic
sc_unsigned a(128), b(128);
sc_unsigned sum = a + b;    // result may need 129 bits
sc_signed diff = a - b;     // difference can be negative -> sc_signed
```

## RTL 對應

```
// Verilog
reg [255:0] wide_addr;
wire [127:0] data_a, data_b;

// SystemC
sc_unsigned wide_addr(256);
sc_unsigned data_a(128), data_b(128);
```

## 相關檔案

- [sc_biguint.md](sc_biguint.md) — 模板子類別 `sc_biguint<W>`
- [sc_signed.md](sc_signed.md) — 有號版本 `sc_signed`
- [sc_nbutils.md](sc_nbutils.md) — 底層向量運算
- [sc_nbdefs.md](sc_nbdefs.md) — 基本型別定義
