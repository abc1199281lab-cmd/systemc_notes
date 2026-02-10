# 2026-02-08-datatypes-bit-int

## 1. SystemC 資料型別概論
SystemC 提供了一套專門為硬體建模設計的資料型別，涵蓋從單一位元到任意精度整數的需求。這些型別位於 `sc_dt` 命名空間中。

## 2. 位元型別 (Bit Types)

### sc_bit - 單一位元 (已棄用)
```cpp
class sc_bit {
    bool m_val;  // 內部儲存
};
```
- **取值**: `0` 或 `1` (對應 `false`/`true`)。
- **狀態**: 自 SystemC 2.1 起標記為 **Deprecated**。
- **建議**: 直接使用 C++ 的 `bool` 型別。

### sc_logic - 四態邏輯 (Four-State Logic)
```cpp
enum sc_logic_value_t {
    Log_0 = 0,  // 邏輯 0
    Log_1 = 1,  // 邏輯 1
    Log_Z = 2,  // 高阻態 (High Impedance)
    Log_X = 3   // 未知態 (Unknown)
};
```
- **必要性**: 模擬真實硬體中的三態匯流排 (Tri-State Bus) 與未初始化狀態。
- **解析函數 (Resolution)**: 當多個驅動源連接到同一訊號時，需進行邏輯解析 (如強制 0/1 優先於 Z，衝突產生 X)。

## 3. 固定位寬整數 (Fixed-Precision Integers)

### sc_int<W> / sc_uint<W>
```cpp
template<int W>
class sc_int : public sc_int_base {
    // W: 位元寬度，範圍 1 至 64
};
```
- **儲存**: 使用原生 64 位元整數 (`int64_t` / `uint64_t`)，效能極高。
- **截斷與符號擴展**: 寫入時自動對 `W` 位元進行掩碼處理，讀取時進行符號擴展。
- **適用場景**: RTL 級建模中的計數器、位址、資料匯流排 (通常 < 64 位元)。

### 與 C++ 原生型別對比
| 型別 | 位元寬 | 運算速度 | 適用場景 |
|------|-------|---------|---------|
| `int` | 平台相關 (通常 32) | 最快 | 控制邏輯、迴圈索引 |
| `sc_int<32>` | 精確 32 | 快 | 硬體訊號、綜合 |
| `sc_bigint<128>` | 任意 | 較慢 | 加密運算、大數運算 |

## 4. 任意精度整數 (Arbitrary-Precision Integers)

### sc_bigint<W> / sc_biguint<W>
- **內部實作**: 使用動態分配的位元陣列 (類似大數運算庫)。
- **無限制**: 位元寬 `W` 可以超過 64，甚至數千位元。
- **運算**: 以軟體模擬方式執行加減乘除，速度較慢但精確。

## 5. 效能考量與選擇建議

### 編譯器優化
- **小位元寬 (<64)**: 編譯器可將 `sc_int` 運算優化為單一機器指令。
- **大位元寬**: `sc_bigint` 需呼叫函數庫，每個運算涉及多次記憶體存取。

### 建模建議
1. **優先使用 `sc_int`**: 當位元寬確定且 < 64 時。
2. **僅必要時使用 `sc_bigint`**: 如密碼學演算法、浮點數運算單元模型。
3. **避免混用**: `sc_int` 與 `sc_bigint` 間的轉換會產生額外開銷。

## 6. 原始碼參考
- `src/sysc/datatypes/bit/sc_bit.h`, `sc_logic.h`: 位元與邏輯型別。
- `src/sysc/datatypes/int/sc_int.h`, `sc_uint.h`: 固定位寬整數。
- `src/sysc/datatypes/int/sc_bigint.h`: 任意精度整數。

---
*Source: ref/systemc/src/sysc/datatypes/*
