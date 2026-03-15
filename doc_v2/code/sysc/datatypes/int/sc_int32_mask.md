# sc_int32_mask.cpp — 32 位元遮罩查找表

## 概述

`sc_int32_mask.cpp` 初始化了 32 位元平台上的 `mask_int` 查找表。這個表用於 `sc_int` 和 `sc_uint` 的部分選取（part-selection）操作，提供預先計算好的位元遮罩，避免執行期的重複計算。

**源檔案：**
- `ref/systemc/src/sysc/datatypes/int/sc_int32_mask.cpp`

**注意：** 此檔案只在 32 位元平台（`_32BIT_`）上編譯。64 位元平台使用 [sc_int64_mask.cpp](sc_int64_mask.md)。

## 日常類比

想像你是一位裁縫，經常需要把布料剪成各種尺寸。與其每次都拿尺量，不如做一套「裁切模板」——1 公分的、2 公分的、3 公分的...需要哪個尺寸直接拿模板來用。

`mask_int` 就是這套「裁切模板」，針對每種可能的位元範圍 `[i:j]` 預先計算好遮罩值。

## 資料結構

```cpp
const uint_type mask_int[SC_INTWIDTH][SC_INTWIDTH];
```

在 32 位元平台上，`SC_INTWIDTH` = 32，所以這是一個 32x32 的二維陣列。

`mask_int[i][j]` 的值是：將第 `j` 位到第 `i` 位之外的位元全部設為 1 的遮罩。用於提取或修改指定範圍的位元。

### 範例

```
mask_int[0][0] = 0xFFFFFFFE  // bit 0 cleared
mask_int[1][0] = 0xFFFFFFFC  // bits 1:0 cleared
mask_int[1][1] = 0xFFFFFFFD  // bit 1 cleared
mask_int[2][0] = 0xFFFFFFF8  // bits 2:0 cleared
```

## 設計原理

### 查表 vs. 即時計算

查表方式犧牲了少量記憶體（32x32 = 1024 個 32 位元整數 = 4KB），換取了部分選取操作的 O(1) 時間複雜度。這在硬體模擬中非常值得，因為位元操作是最頻繁的操作之一。

## 相關檔案

- [sc_int64_mask.md](sc_int64_mask.md) — 64 位元版本
- [sc_int_base.md](sc_int_base.md) — 使用此遮罩表的類別
- [sc_uint_base.md](sc_uint_base.md) — 使用此遮罩表的類別
