# sc_string - 字串工具與數字表示法

## 概述

`sc_string.h` 提供了數字表示法（number representation）的列舉定義和 I/O 串流輔助函式。儘管檔案名稱叫「sc_string」，它現在的主要內容是 `sc_numrep` 列舉和相關的 I/O 工具，而不是一個字串類別實作。

**來源檔案**：`sysc/utils/sc_string.h`（僅標頭檔）

## 生活比喻

當你要告訴別人一個數字，可以用不同的方式：
- 十進位：255
- 十六進位：0xFF
- 二進位：11111111
- 八進位：0377

`sc_numrep` 就是列舉了所有 SystemC 支援的數字表示法，讓程式知道該用哪種方式來顯示數值。

## sc_numrep 列舉

```cpp
enum sc_numrep {
    SC_NOBASE = 0,    // 無基底
    SC_BIN    = 2,    // 二進位
    SC_OCT    = 8,    // 八進位
    SC_DEC    = 10,   // 十進位
    SC_HEX    = 16,   // 十六進位
    SC_BIN_US,        // 二進位無號
    SC_BIN_SM,        // 二進位符號大小
    SC_OCT_US,        // 八進位無號
    SC_OCT_SM,        // 八進位符號大小
    SC_HEX_US,        // 十六進位無號
    SC_HEX_SM,        // 十六進位符號大小
    SC_CSD            // 典型符號數字 (Canonical Signed Digit)
};
```

注意 `SC_BIN`、`SC_OCT`、`SC_DEC`、`SC_HEX` 的值就是對應的基底數字（2、8、10、16），方便直接用於計算。

### US 與 SM 後綴

- **US (Unsigned)**：將位元模式視為無號數
- **SM (Sign-Magnitude)**：最高位元為符號位，其餘為大小

## I/O 輔助函式

### sc_io_base — 取得串流的基底

```cpp
inline sc_numrep sc_io_base(systemc_ostream& stream, sc_numrep def_base);
```

根據串流的格式旗標（`std::ios::basefield`）回傳對應的 `sc_numrep`：
- `std::ios::dec` → `SC_DEC`
- `std::ios::hex` → `SC_HEX`
- `std::ios::oct` → `SC_OCT`
- 其他 → `def_base`（預設值）

### sc_io_show_base — 是否顯示基底前綴

```cpp
inline bool sc_io_show_base(systemc_ostream& stream);
```

回傳串流是否設定了 `std::ios::showbase` 旗標（例如是否要顯示 `0x` 前綴）。

## 向後相容

```cpp
#ifdef SC_USE_STD_STRING
typedef ::std::string sc_string;
#endif
```

若定義了 `SC_USE_STD_STRING`，`sc_string` 會被定義為 `std::string` 的別名，用於舊程式碼的相容。

## 命名空間

注意 `sc_numrep` 和相關函式定義在 `sc_dt` 命名空間中（不是 `sc_core`），因為它們與資料型別（data types）有關。

## 相關檔案

- [sc_string_view.md](sc_string_view.md) — 不擁有記憶體的字串參照
- [sc_iostream.md](sc_iostream.md) — I/O 串流標頭包裝
