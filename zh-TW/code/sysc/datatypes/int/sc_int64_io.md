# sc_int64_io.cpp — 64 位元整數 I/O 支援

## 概述

`sc_int64_io.cpp` 提供了 64 位元整數的輸入/輸出支援函式，特別是為了解決 Microsoft Visual C++ 編譯器（MSVC）在處理 `uint64` 和 `int64` 型別的 `<<` 運算子時的相容性問題。

**源檔案：**
- `ref/systemc/src/sysc/datatypes/int/sc_int64_io.cpp`

## 日常類比

這就像翻譯服務。某些老式印表機（MSVC 編譯器）不認識新的數字格式（`uint64`），所以需要一個翻譯器把數字轉成印表機看得懂的格式（字元字串）再印出來。

## 核心功能

### write_uint64

```cpp
static void write_uint64(::std::ostream& os, uint64 val, int sign);
```

手動將 `uint64` 值轉換為字串並寫入輸出串流。支援：
- **八進位**（octal）格式
- **十六進位**（hex）格式
- **十進位**（decimal）格式
- 前綴顯示（`0x`、`0`）
- 正號顯示（`+`）

### 條件編譯

```cpp
#if defined( _MSC_VER )
// ... MSVC-specific implementations
#endif
```

整個實作被 `_MSC_VER` 巨集包圍，只在 MSVC 編譯器下才會編譯。其他編譯器（GCC、Clang）的標準庫已經正確支援 `uint64` 的 I/O。

## 其他 include

這個檔案也間接觸發了一些重要的 inline 函式展開：

```cpp
#include "sysc/datatypes/int/sc_signed_ops.h"
#include "sysc/datatypes/int/sc_signed_inlines.h"
#include "sysc/datatypes/int/sc_unsigned_inlines.h"
```

這些 include 確保了需要多個標頭檔完整定義後才能編譯的函式被正確實例化。

## 相關檔案

- [sc_int_base.md](sc_int_base.md) — 使用 I/O 功能的類別
- [sc_uint_base.md](sc_uint_base.md) — 使用 I/O 功能的類別
- [sc_signed.md](sc_signed.md) — inline 函式在此展開
- [sc_unsigned.md](sc_unsigned.md) — inline 函式在此展開
