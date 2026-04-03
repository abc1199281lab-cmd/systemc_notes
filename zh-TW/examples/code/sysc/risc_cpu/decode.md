# Decode -- 指令解碼單元

## 軟體類比

Decode 單元就像一個命令解析器 (command parser)。想像你在實作一個 CLI 工具或腳本直譯器：

```python
# 軟體類比
def decode(raw_instruction):
    opcode = (raw_instruction >> 24) & 0xFF       # 取出操作碼
    regC   = (raw_instruction >> 20) & 0xF        # 目標暫存器
    regA   = (raw_instruction >> 16) & 0xF        # 來源暫存器 A
    regB   = (raw_instruction >> 12) & 0xF        # 來源暫存器 B
    imm    = raw_instruction & 0xFFFF             # 立即值

    match opcode:
        case 0x01: return ("ADD", regC, regs[regA], regs[regB])
        case 0x04: return ("SUB", regC, regs[regA], regs[regB])
        case 0x10: return ("BEQ", regC, regA, imm)
        ...
```

Decode 是這個 CPU 中**最複雜的模組**，因為它同時負責：解碼指令、讀取暫存器、接收寫回資料、以及分支判斷。

## 原始檔案

- `decode.h` -- 模組宣告
- `decode.cpp` -- 行為實作

## 指令格式

每條指令是 32-bit unsigned integer，格式如下：

```
[31:24]  opcode       (8 bits)  -- 操作碼
[23:20]  regC         (4 bits)  -- 目標暫存器
[19:16]  regA         (4 bits)  -- 來源暫存器 A
[15:12]  regB         (4 bits)  -- 來源暫存器 B
[15:0]   immediate    (16 bits) -- 立即值 (與 regB 重疊)
[11:0]   offset       (12 bits) -- 記憶體偏移量
[23:0]   longlabel    (24 bits) -- 長跳轉標籤
```

## 指令集總覽

### 整數運算 (送往 ALU)

| Opcode | 指令 | 說明 | ALU Op |
|--------|------|------|--------|
| 0x00 | HALT | 暫停並傾印暫存器 | - |
| 0x01 | ADD Rc, Ra, Rb | Rc = Ra + Rb | 3 |
| 0x02 | ADDI Rc, Ra, #imm | Rc = Ra + imm | 3 |
| 0x03 | ADDC Rc, Ra, Rb | Rc = Ra + Rb + Carry | 1 |
| 0x04 | SUB Rc, Ra, Rb | Rc = Ra - Rb | 4 |
| 0x05 | SUBI Rc, Ra, #imm | Rc = Ra - imm | 4 |
| 0x06 | SUBC Rc, Ra, Rb | Rc = Ra - Rb - Carry | 2 |
| 0x07 | MUL Rc, Ra, Rb | Rc = Ra * Rb | 5 |
| 0x08 | DIV Rc, Ra, Rb | Rc = Ra / Rb | 6 |
| 0x09 | NAND Rc, Ra, Rb | Rc = ~(Ra & Rb) | 7 |
| 0x0A | AND Rc, Ra, Rb | Rc = Ra & Rb | 8 |
| 0x0B | OR Rc, Ra, Rb | Rc = Ra \| Rb | 9 |
| 0x0C | XOR Rc, Ra, Rb | Rc = Ra ^ Rb | 10 |
| 0x0D | NOT Rc, Ra | Rc = ~Ra | 11 |
| 0x0E | MOD Rc, Ra, Rb | Rc = Ra % Rb | 14 |
| 0x0F | MOV Rc, Ra | Rc = Ra | 3 |
| 0xF1 | MOVI Rc, #imm | Rc = imm | 3 |

### 分支指令

| Opcode | 指令 | 說明 |
|--------|------|------|
| 0x10 | BEQ Rc, Ra, label | if Rc == Ra then PC += label |
| 0x11 | BNE Rc, Ra, label | if Rc != Ra then PC += label |
| 0x12 | BGT Rc, Ra, label | if Rc > Ra then PC += label |
| 0x13 | BGE Rc, Ra, label | if Rc >= Ra then PC += label |
| 0x14 | BLT Rc, Ra, label | if Rc < Ra then PC += label |
| 0x15 | BLE Rc, Ra, label | if Rc <= Ra then PC += label |
| 0x16 | J longlabel | 無條件跳轉 |

### 記憶體操作

| Opcode | 指令 | 說明 |
|--------|------|------|
| 0x4D | LW Rc, Ra, offset | Rc = mem[Ra + offset] |
| 0x4E | SW Rc, Ra, offset | mem[Ra + offset] = Rc |

### 浮點運算 (送往 FPU)

| Opcode | 指令 | 說明 |
|--------|------|------|
| 0x29 | FADD | 浮點加法 |
| 0x2A | FSUB | 浮點減法 |
| 0x2B | FMUL | 浮點乘法 |
| 0x2C | FDIV | 浮點除法 |

### MMX/SIMD 運算 (送往 MMX Unit)

| Opcode | 指令 | 說明 |
|--------|------|------|
| 0x31 | PADD | Packed byte 加法 |
| 0x32 | PADDS | Packed byte 飽和加法 |
| 0x33 | PSUB | Packed byte 減法 |
| 0x34 | PSUBS | Packed byte 飽和減法 |
| 0x35 | PMADD | Packed multiply-add |
| 0x36 | PACK | 資料打包 |
| 0x37 | MMXCK | Chroma keying |

### 系統指令

| Opcode | 指令 | 說明 |
|--------|------|------|
| 0xE0 | FLUSH | 清除所有暫存器 |
| 0xF0 | LDPID | 載入 Process ID |
| 0xFF | QUIT | 結束模擬 |

## 內部狀態

```cpp
signed int cpu_reg[32];       // 32 個通用暫存器 (類似 HashMap<int, int>)
signed int vcpu_reg[32];      // 虛擬暫存器 (未使用)
bool cpu_reg_lock[32];        // 暫存器鎖定旗標 (用於 data hazard)
unsigned int pc_reg;          // Program Counter
unsigned int jalpc_reg;       // 返回位址暫存器 (用於函式呼叫)
```

暫存器檔案在建構時從 `register.img` 載入初始值。

## 行為邏輯

### 寫回處理 (Writeback)

每個時脈週期開始時，Decode 先檢查是否有資料需要寫回暫存器：

1. **ALU 結果寫回**：若 `destreg_write == true`，將 ALU 輸出寫入 `cpu_reg[destreg_write_src]`
2. **記憶體載入寫回**：若 `dram_rd_valid == true`，將 DCache 資料寫入暫存器
3. **FPU/MMX 結果寫回**：若 `fpu_valid == true`，將浮點/MMX 結果寫入暫存器

這就像一個 event handler 在每個 tick 先處理「回覆」再處理「新請求」。

### 解碼與分派

讀取指令後，透過 bit masking 萃取各欄位，然後用一個大型 `switch` 語句根據 opcode 決定動作：

- **ALU 指令**：設定 `src_A`, `src_B`, `alu_op`, `decode_valid=true`，送往 Execute
- **FPU 指令**：設定相同運算元，但啟用 `float_valid=true`
- **MMX 指令**：設定相同運算元，但啟用 `mmx_valid=true`
- **分支指令**：在 Decode 階段直接比較暫存器值，若分支成立則設定 `branch_valid=true` 和 `branch_target_address`
- **記憶體指令**：設定 `mem_access=true` 和 `mem_address`

### 分支處理

分支判斷在 Decode 階段完成（而非 Execute），這減少了分支延遲。向後分支（backward branch）透過 bit 15 的符號位來判斷，並將 label 轉換為負數偏移量。

## SystemC 重點

- 32 個暫存器以 C++ array 實作，不需要 `sc_signal`，因為它們是模組的私有狀態。
- 多個寫回來源（ALU、DCache、FPU）在同一個 process 中循序處理，避免了多重寫入衝突。
- `sc_stop()` 在收到 QUIT (0xFF) 指令時被呼叫，終止整個模擬。
