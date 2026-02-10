# 2026-02-08-datatypes-fixed

## 1. 定點數 (Fixed-Point) 概論
定點數是 SystemC 為 DSP 與嵌入式系統建模提供的資料型別，用於模擬沒有硬體浮點單元 (FPU) 的系統。透過整數部分 (Integer Word Length) 與分數部分 (Fractional Word Length) 的明確劃分，實現確定性的數值表示。

## 2. 定點數型別

### sc_fix / sc_ufix
```cpp
sc_fix( int wl, int iwl, sc_q_mode qm, sc_o_mode om, int nb );
// wl: 總位元寬 (Word Length)
// iwl: 整數位元寬 (Integer Word Length)
// qm: 量化模式 (Quantization Mode)
// om: 溢位模式 (Overflow Mode)
// nb: 溢位位元數 (Number of Bits)
```
- **sc_fix**: 有號定點數（Two's Complement）。
- **sc_ufix**: 無號定點數。

### 數值範圍計算
對於 `sc_fix<wl, iwl>`:
- **分數位元**: `fwl = wl - iwl`
- **解析度 (Resolution)**: $2^{-fwl}$
- **範圍**:
  - 有號: $[-2^{iwl-1}, 2^{iwl-1} - 2^{-fwl}]$
  - 無號: $[0, 2^{iwl} - 2^{-fwl}]$

**範例**: `sc_fix<16, 8>`
- 整數位元: 8 (含 1 位元符號)
- 分數位元: 8
- 解析度: $1/256 \approx 0.0039$
- 範圍: $-128$ 到 $127.996$

## 3. 量化模式 (Quantization Modes)
當數值精度超過分數位元表示能力時，需進行量化處理：

| 模式 | 名稱 | 行為 |
|------|------|------|
| `SC_RND` | 四捨五入 | 最接近值，中間値向上 |
| `SC_RND_ZERO` | 向零捨入 | 截斷 (Truncate) |
| `SC_RND_MIN_INF` | 向負無窮 | Floor |
| `SC_RND_INF` | 向無窮遠離零 | 絕對值變大 |
| `SC_RND_CONV` | 收斂式捨入 | 偶數優先 (避免偏置) |
| `SC_TRN` | 截斷 | 預設，最快但精度最差 |
| `SC_TRN_ZERO` | 截斷向零 | 對稱截斷 |

## 4. 溢位模式 (Overflow Modes)
當數值超出表示範圍時的處理策略：

| 模式 | 名稱 | 行為 |
|------|------|------|
| `SC_WRAP` | 環繞 | 模運算，數值回繞 (不建議) |
| `SC_SAT` | 飽和 | 限制在最大/最小值 (最常用) |
| `SC_SAT_ZERO` | 飽和到零 | 溢位時歸零 |
| `SC_SAT_SYM` | 對稱飽和 | 最小值為 $-max$ 而非 $-max-1$ |
| `SC_WRAP_SM` | 符號量值環繞 | 保留符號位元 |

## 5. 效能考量：sc_fix vs sc_fix_fast

| 型別 | 底層實作 | 精度 | 速度 | 適用場景 |
|------|---------|------|------|---------|
| `sc_fix` | 任意精度運算 | 無限制 | 較慢 | 演算法驗證 |
| `sc_fix_fast` | `double` | ~53 位元 | 快 | 快速模擬 |

**建議**:
- 開發初期使用 `sc_fix` 確保精度正確。
- 驗證後改用 `sc_fix_fast` 加速模擬。

## 6. 原始碼參考
- `src/sysc/datatypes/fx/sc_fix.h`: 無限制定點數類別。
- `src/sysc/datatypes/fx/sc_fix_fast.h`: 快速定點數 (double 為基礎)。
- `src/sysc/datatypes/fx/sc_fxnum.h`: 定點數基類與量化/溢位處理。

---
*Source: ref/systemc/src/sysc/datatypes/fx/*
