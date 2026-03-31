# MMX Unit -- SIMD 執行單元

## 軟體類比

MMX 單元實作了 SIMD (Single Instruction, Multiple Data) 運算。如果你用過 numpy，你已經理解 SIMD 的核心概念：

```python
# 軟體類比：一般的逐元素運算
for i in range(4):
    c[i] = a[i] + b[i]

# SIMD 等價（numpy 風格）
c = a + b   # 一次操作，四個元素同時計算
```

在硬體中，MMX 把一個 32-bit 暫存器拆成四個 8-bit 的 byte，對每個 byte 獨立執行相同運算。這在影像處理和多媒體應用中非常有用 -- 例如對四個像素值同時做加法。

## 原始檔案

- `mmxu.h` -- 模組宣告
- `mmxu.cpp` -- 行為實作

## 模組介面

| 方向 | 信號名稱 | 類型 | 說明 |
|------|-----------|------|------|
| 輸入 | `mmx_valid` | `sc_in<bool>` | MMX 啟用 |
| 輸入 | `opcode` | `sc_in<int>` | 操作碼 |
| 輸入 | `mmxa` | `sc_in<signed int>` | 運算元 A (packed 4 bytes) |
| 輸入 | `mmxb` | `sc_in<signed int>` | 運算元 B (packed 4 bytes) |
| 輸入 | `dest` | `sc_in<unsigned>` | 目標暫存器 |
| 輸出 | `mmxdout` | `sc_out<signed int>` | 運算結果 |
| 輸出 | `mmxout_valid` | `sc_out<bool>` | 輸出有效 |
| 輸出 | `mmxdestout` | `sc_out<unsigned>` | 目標暫存器編號 |

## Packed Byte 格式

一個 32-bit 值被拆為四個 8-bit lanes：

```
[31:24] byte3 (a3)    [23:16] byte2 (a2)    [15:8] byte1 (a1)    [7:0] byte0 (a0)
```

這等同於軟體中的：

```python
a3 = (value >> 24) & 0xFF
a2 = (value >> 16) & 0xFF
a1 = (value >>  8) & 0xFF
a0 =  value        & 0xFF
```

## 支援的操作

| Op Code | 指令 | 說明 | 軟體類比 |
|---------|------|------|----------|
| 3 | PADD | Packed byte 加法 | `np.add(a, b)` (uint8) |
| 4 | PADDS | 飽和加法（上限 255） | `np.clip(a + b, 0, 255)` |
| 5 | PSUB | Packed byte 減法 | `np.subtract(a, b)` |
| 6 | PSUBS | 飽和減法（下限 0） | `np.clip(a - b, 0, 255)` |
| 7 | PMADD | Packed multiply-add | `a3*b3+a2*b2` 和 `a1*b1+a0*b0` |
| 8 | PACK | 16-bit to 8-bit 打包 | 資料壓縮 |
| 9 | MMXCK | Chroma Keying | 比較遮罩（用於綠幕去背） |

### 飽和運算 (Saturation)

一般加法溢出時會 wrap around（例如 200 + 100 = 44，因為 300 mod 256 = 44）。飽和運算則會 clamp 到最大值 255，這在處理像素值時很重要，避免顏色溢出導致視覺錯誤。

### Chroma Keying

`MMXCK` 操作比較兩個像素的每個 byte channel 是否相等，相等輸出 0xFF，不相等輸出 0x00。這是影片合成中「綠幕去背」的基本操作 -- 比較每個像素是否為特定的背景顏色。

## 時序特性

- 所有 MMX 操作都只需要 1 個時脈週期
- 輸出信號 `mmxdout` 與 FPU 的 `fdout` 共用（透過 `SC_MANY_WRITERS`），因為 Decode 保證不會同時啟用 FPU 和 MMX

## SystemC 重點

- MMX 和 FPU 寫入相同的輸出信號（`fdout`, `fout_valid`, `fdestout`），使用 `SC_MANY_WRITERS` 寫入策略來允許多個 writer。
- 在 `main.cpp` 中可以看到 MMX 的輸出直接連接到與 FPU 相同的信號線，Decode 透過 `fpu_valid` 信號統一接收兩者的結果。
