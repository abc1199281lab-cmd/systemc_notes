# tlm_helpers.h - 端序偵測輔助函式

## 概述

`tlm_helpers.h` 提供了幾個簡單的輔助函式，用於在執行時偵測主機（host）的位元組順序（endianness）。這些函式在端序轉換和跨平台模擬中扮演基礎角色。

## 日常類比

就像出國旅遊前先確認目的地的電壓是 110V 還是 220V——這些函式幫你「偵測」你的電腦是用哪種位元組排列方式，以便決定是否需要做「電壓轉換」（端序轉換）。

## 列舉

```cpp
enum tlm_endianness {
  TLM_UNKNOWN_ENDIAN,   // not yet determined
  TLM_LITTLE_ENDIAN,    // least significant byte first (x86, ARM default)
  TLM_BIG_ENDIAN        // most significant byte first (PowerPC, SPARC)
};
```

## 函式詳情

### `get_host_endianness()`

```cpp
inline tlm_endianness get_host_endianness(void);
```

回傳主機的端序。使用靜態變數做快取——只在第一次呼叫時偵測。

**偵測原理：**
```cpp
unsigned int number = 1;
unsigned char* p = (unsigned char*)&number;
// if p[0] == 0 -> big endian (MSB first)
// if p[0] == 1 -> little endian (LSB first)
```

整數 `1` 在記憶體中的排列：
- Little-endian: `[0x01, 0x00, 0x00, 0x00]` -- 最低位在前
- Big-endian: `[0x00, 0x00, 0x00, 0x01]` -- 最高位在前

### `host_has_little_endianness()`

```cpp
inline bool host_has_little_endianness(void);
```

回傳 `true` 表示主機是 little-endian。同樣使用靜態變數快取。

### `has_host_endianness(endianness)`

```cpp
inline bool has_host_endianness(tlm_endianness endianness);
```

檢查給定的端序是否與主機相同。常用於決定是否需要做端序轉換：

```cpp
if (!has_host_endianness(target_endianness)) {
  // need endian conversion
  tlm_to_hostendian_generic<uint32_t>(&txn, bus_width);
}
```

## 原始碼位置

`ref/systemc/src/tlm_core/tlm_2/tlm_generic_payload/tlm_helpers.h`

## 相關檔案

- [tlm_endian_conv.md](tlm_endian_conv.md) - 端序轉換函式
- [tlm_generic_payload.md](tlm_generic_payload.md) - 通用酬載
