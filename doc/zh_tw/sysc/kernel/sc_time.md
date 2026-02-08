# SystemC Time 分析

> **檔案**: `ref/systemc/src/sysc/kernel/sc_time.cpp`

## 1. 概述
`sc_time` 代表模擬時間。在核心中，時間不是浮點數，而是 "時間步 (time steps)" 的 **64-bit 無號整數** 計數。

## 2. 核心概念

### 2.1 時間解析度 (`sc_time_params`)
SystemC 強制執行一個全域時間解析度 (最小可表示時間單位)。
- **預設**: 1 ps (`1e-12` s)。
- **儲存**: `sc_time` 中的 `m_value` 是這些解析度步數的計數。
- **範例**: 如果解析度是 1ns，`sc_time(10, SC_NS)` 儲存 `10`。如果解析度是 1ps，它儲存 `10,000`。

### 2.2 建構與轉換
當你寫 `sc_time(10.5, SC_NS)`：
1.  它取得全域時間解析度。
2.  它將 `10.5 ns` 轉換為解析度步數。
3.  **四捨五入 (Rounding)**: 它四捨五入到最接近的整數。`10.5 * scaling_factor + 0.5`。

### 2.3 `time_params` 生命週期
- `sc_time_params` 是 `sc_simcontext` 中的單例結構。
- **鎖定 (Locking)**: 一旦 *第一個* `sc_time` 物件被建立，解析度就被 "凍結" (`time_resolution_fixed`)。
    - *暗示*: 你必須在宣告任何 `sc_time` 常數 *之前* 呼叫 `sc_set_time_resolution()`。

### 2.4 字串轉換
存在解析如 `"10 ns"`, `"5.5 us"` 字串的邏輯。這通常用於讀取設定檔或命令列參數。

---

## 3. 硬體 / RTL 對應

| SystemC (`sc_time`) | Verilog / SystemVerilog |
| :--- | :--- |
| `sc_time` | `#` 延遲值 |
| `sc_set_time_resolution()` | `timescale` (精度部分) |
| `sc_start(time)` | `initial #time $finish;` |

- **精度**: 不像 Verilog `timescale` 可以每個模組不同，SystemC 有一個 **單一全域解析度**。這防止了在連接不同 timescale 模組時出現的經典 Verilog 精度遺失問題。

---

## 4. 關鍵重點
1.  **整數運算**: 在內部，時間是 `uint64`。這避免了長時間模擬中的浮點數漂移。
2.  **全域解析度**: 儘早設定一次。如果你需要飛秒 (femtosecond) 精度，在建立任何物件之前將其設定為 `SC_FS`。
3.  **效能**: 使用 `SC_ZERO_TIME` 通常會最佳化為特定的檢查，但建立許多 `sc_time` 物件 (轉換) 有很小的成本。
