# Floating Point Unit -- 浮點執行單元

## 軟體類比

FPU 就像一個專門處理浮點運算的協處理器，相當於軟體中引入一個專用的數學運算函式庫。在 CPU 還沒有整合 FPU 的年代，浮點運算需要透過獨立的硬體晶片完成 -- 就像你的 web app 把密集計算委派給一個 Worker thread 或微服務：

```python
# 軟體類比
class FloatCoprocessor:
    def compute(self, op, a, b):
        # 拆解 IEEE 754 格式
        sign_a, exp_a, mant_a = self.unpack(a)
        sign_b, exp_b, mant_b = self.unpack(b)
        # 對齊指數
        mant_a, mant_b, exp = self.align(exp_a, mant_a, exp_b, mant_b)
        # 執行運算
        result_mant = self.operate(op, mant_a, mant_b)
        # 重新打包
        return self.pack(sign_a, exp, result_mant)
```

## 原始檔案

- `floating.h` -- 模組宣告
- `floating.cpp` -- 行為實作

## 模組介面

| 方向 | 信號名稱 | 類型 | 說明 |
|------|-----------|------|------|
| 輸入 | `in_valid` | `sc_in<bool>` | FPU 啟用 |
| 輸入 | `opcode` | `sc_in<int>` | 操作碼 |
| 輸入 | `floata` | `sc_in<signed int>` | 運算元 A (IEEE 754 格式) |
| 輸入 | `floatb` | `sc_in<signed int>` | 運算元 B (IEEE 754 格式) |
| 輸入 | `dest` | `sc_in<unsigned>` | 目標暫存器編號 |
| 輸出 | `fdout` | `sc_out<signed int>` | 運算結果 |
| 輸出 | `fout_valid` | `sc_out<bool>` | 輸出有效 |
| 輸出 | `fdestout` | `sc_out<unsigned>` | 目標暫存器編號 |

## IEEE 754 浮點格式

FPU 手動拆解和組裝 IEEE 754 single-precision 格式：

```
[31]     sign        (1 bit)   -- 符號位 (0=正, 1=負)
[30:23]  exponent    (8 bits)  -- 指數
[22:0]   significand (23 bits) -- 有效數字 (mantissa)
```

在軟體中，你通常直接使用 `float` 或 `double` 型態，IEEE 754 的細節被語言隱藏了。但在硬體中，FPU 必須自己實作每一步。

## 運算流程

1. **拆解 (Unpack)**：從 32-bit 整數中萃取 sign、exponent、significand
2. **指數對齊 (Alignment)**：將較小指數的 significand 右移，使兩個運算元的指數相同
3. **運算 (Compute)**：對齊後的 significand 進行加/減/乘/除
4. **正規化 (Normalize)**：處理溢出，重新打包為 IEEE 754 格式

### 支援的操作

| Op Code | 操作 | 說明 |
|---------|------|------|
| 0 | Stall | 保持前一個值 |
| 3 | FADD | 浮點加法 |
| 4 | FSUB | 浮點減法 |
| 5 | FMUL | 浮點乘法（指數加倍） |
| 6 | FDIV | 浮點除法 |

## 限制

原始碼註解中明確說明了以下限制：
- 只有在兩個運算元的符號位相同時才能正確運算
- 溢出處理被簡化（overflow is ignored）

這些限制在一個教學範例中是可以接受的 -- 完整的 IEEE 754 實作會複雜許多。

## 時序特性

- 初始化等待 3 個時脈週期
- 等待 `in_valid` 信號
- 指數對齊花費 1 個時脈週期
- 實際運算花費 1 個時脈週期
- 輸出結果後等待 1 個時脈週期才清除 valid 信號

整體一次浮點運算大約需要 4-5 個時脈週期，反映了浮點運算比整數運算慢的現實。

## SystemC 重點

- 使用 `do { wait(); } while (!(in_valid == true))` 模式等待輸入，這是一個常見的 SystemC 同步模式。
- FPU 和 ALU 是平行的執行單元，由 Decode 透過不同的 valid 信號選擇啟用哪一個。
- 輸出信號 `fdout`, `fout_valid`, `fdestout` 與 MMX 單元共用（`SC_MANY_WRITERS` policy），因為兩者不會同時運作。
