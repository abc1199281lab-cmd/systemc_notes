# Execute -- 整數執行單元 (ALU)

## 軟體類比

Execute 單元就是 `eval()` 函式。它接收已經解析好的操作碼和運算元，然後執行對應的運算：

```python
# 軟體類比
def alu_execute(opcode, a, b):
    match opcode:
        case 3:  return a + b
        case 4:  return a - b
        case 5:  return a * b
        case 6:  return a // b
        case 7:  return ~(a & b)
        case 8:  return a & b
        case 9:  return a | b
        case 10: return a ^ b
        case 11: return ~a
        case 14: return a % b
```

就是這麼直接。複雜的部分在 Decode，Execute 只負責純粹的計算。

## 原始檔案

- `exec.h` -- 模組宣告
- `exec.cpp` -- 行為實作

## 模組介面

### 輸入信號

| 信號名稱 | 類型 | 說明 |
|-----------|------|------|
| `in_valid` | `sc_in<bool>` | Decode 輸出有效 |
| `opcode` | `sc_in<int>` | ALU 操作碼 |
| `dina` | `sc_in<signed int>` | 運算元 A |
| `dinb` | `sc_in<signed int>` | 運算元 B |
| `dest` | `sc_in<unsigned>` | 目標暫存器編號 |
| `forward_A` / `forward_B` | `sc_in<bool>` | 資料轉發旗標（未使用） |

### 輸出信號

| 信號名稱 | 類型 | 說明 |
|-----------|------|------|
| `dout` | `sc_out<signed int>` | 運算結果 |
| `out_valid` | `sc_out<bool>` | 輸出有效 |
| `destout` | `sc_out<unsigned>` | 目標暫存器編號（傳遞給 Decode 寫回） |
| `C` | `sc_out<bool>` | Carry flag（進位旗標） |
| `V` | `sc_out<bool>` | Overflow flag（溢位旗標） |
| `Z` | `sc_out<bool>` | Zero flag（零旗標） |

## 支援的操作

| Op Code | 操作 | 延遲 (clock cycles) | 說明 |
|---------|------|---------------------|------|
| 0 | Stall | 1 | 保持前一個輸出值 |
| 1 | Add + Carry | 1 | a + b + carry |
| 2 | Sub + Carry | 1 | a - b - carry |
| 3 | Add | 1 | a + b |
| 4 | Sub | 1 | a - b |
| 5 | Multiply | 2 | a * b（多一個週期） |
| 6 | Divide | 2 | a / b（多一個週期，含除零檢查） |
| 7 | NAND | 1 | ~(a & b) |
| 8 | AND | 1 | a & b |
| 9 | OR | 1 | a \| b |
| 10 | XOR | 1 | a ^ b |
| 11 | NOT | 1 | ~a |
| 12 | Left Shift | 1 | a << b |
| 13 | Right Shift | 1 | a >> b |
| 14 | Modulo | 1 | a % b |

## 狀態旗標 (Condition Flags)

每次運算完成後，ALU 更新三個狀態旗標：

- **Z (Zero)**：結果為零時設定。類似 `if (result == 0)` 的 boolean 狀態。
- **C (Carry)**：結果的第 32 bit 被設定時觸發，表示無符號溢出。
- **V (Overflow)**：結果超過 32-bit 範圍時觸發，表示有符號溢出。

在軟體中，這些旗標等同於在每次運算後自動更新的全域狀態變數，供後續分支指令使用。

## 時序特性

- 大多數操作花費 1 個時脈週期
- 乘法和除法花費 2 個時脈週期（模擬真實硬體中這些操作較慢的特性）
- `wait(3)` 在迴圈開始前表示 Execute 單元在初始化後等待 3 個週期才開始運作，確保 pipeline 前面的階段已經準備好資料

## SystemC 重點

- 使用 `sc_dt::int64` 儲存中間結果，以偵測 32-bit 溢出。這類似於在 JavaScript 中使用 BigInt 來偵測 Number 溢出。
- 輸出在一個 `wait()` 後被設為 invalid (`out_valid=false`)，確保 Decode 只看到一個週期的有效信號 -- 這是一個常見的 pulse 信號模式。
