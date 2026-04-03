# sc_int64_mask.cpp — 64 位元遮罩查找表

## 概述

`sc_int64_mask.cpp` 初始化了 64 位元平台上的 `mask_int` 查找表。功能與 `sc_int32_mask.cpp` 相同，但遮罩值是 64 位元的，並且陣列大小是 64x64。

**源檔案：**
- `ref/systemc/src/sysc/datatypes/int/sc_int64_mask.cpp`

## 與 32 位元版本的差異

| 特性 | sc_int32_mask | sc_int64_mask |
|------|---------------|---------------|
| 條件編譯 | `#ifdef _32BIT_` | 無條件（預設） |
| `SC_INTWIDTH` | 32 | 64 |
| 陣列大小 | 32x32 | 64x64 |
| 元素型別 | `uint32_t` | `uint64_t` |
| 記憶體用量 | ~4 KB | ~32 KB |

## 資料結構

```cpp
const uint_type mask_int[SC_INTWIDTH][SC_INTWIDTH];
// SC_INTWIDTH = 64 on 64-bit platforms
```

### 範例值

```
mask_int[0][0] = 0xFFFFFFFFFFFFFFFE  // bit 0 cleared
mask_int[1][0] = 0xFFFFFFFFFFFFFFFC  // bits 1:0 cleared
mask_int[1][1] = 0xFFFFFFFFFFFFFFFD  // bit 1 cleared
```

## 設計原理

現代系統幾乎都是 64 位元，所以這個檔案是實際使用的版本。32 KB 的記憶體換取高速的位元操作，在硬體模擬場景下是非常合理的取捨。

## 相關檔案

- [sc_int32_mask.md](sc_int32_mask.md) — 32 位元版本
- [sc_int_base.md](sc_int_base.md) — 使用此遮罩表
- [sc_uint_base.md](sc_uint_base.md) — 使用此遮罩表
