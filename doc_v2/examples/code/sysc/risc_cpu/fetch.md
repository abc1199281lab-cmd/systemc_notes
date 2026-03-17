# Fetch -- 指令取得單元

## 軟體類比

Fetch 單元的工作就像一個腳本直譯器中「讀取下一行程式碼」的函式。想像你正在實作一個簡單的 bytecode VM：

```python
# 軟體類比：一個簡單的 instruction fetch loop
pc = 0
while True:
    instruction = memory[pc]    # <-- 這就是 Fetch 在做的事
    pc += 1
    if branch_taken:
        pc = branch_target
    if interrupt_pending:
        pc = interrupt_vector
    yield instruction
```

差別在於：硬體版本需要處理記憶體延遲 (memory latency)、快取命中/未命中、以及和其他模組之間的時序同步。

## 原始檔案

- `fetch.h` -- 模組宣告（介面定義）
- `fetch.cpp` -- 行為實作

## 模組介面

### 輸入信號

| 信號名稱 | 類型 | 說明 |
|-----------|------|------|
| `ramdata` | `sc_in<unsigned>` | 從記憶體讀回的指令資料 |
| `branch_address` | `sc_in<unsigned>` | 分支跳轉的目標位址 |
| `next_pc` | `sc_in<bool>` | 是否遞增 PC |
| `branch_valid` | `sc_in<bool>` | 分支是否有效 |
| `stall_fetch` | `sc_in<bool>` | 暫停取指令（流水線暫停） |
| `interrupt` | `sc_in<bool>` | 中斷請求 |
| `int_vectno` | `sc_in<unsigned>` | 中斷向量編號 |
| `bios_valid` | `sc_in<bool>` | BIOS 資料就緒 |
| `icache_valid` | `sc_in<bool>` | ICache 資料就緒 |
| `pred_fetch` | `sc_in<bool>` | 分支預測觸發 |
| `pred_branch_address` | `sc_in<unsigned>` | 預測的分支目標位址 |
| `pred_branch_valid` | `sc_in<bool>` | 分支預測是否有效 |

### 輸出信號

| 信號名稱 | 類型 | 說明 |
|-----------|------|------|
| `ram_cs` | `sc_out<bool>` | 記憶體 chip select（啟用記憶體） |
| `ram_we` | `sc_out<bool>` | 記憶體寫入致能 |
| `address` | `sc_out<unsigned>` | 送往記憶體的位址 |
| `instruction` | `sc_out<unsigned>` | 送往 Decode 的指令 |
| `instruction_valid` | `sc_out<bool>` | 指令有效 |
| `program_counter` | `sc_out<unsigned>` | 目前的 PC 值 |
| `interrupt_ack` | `sc_out<bool>` | 中斷確認回應 |
| `branch_clear` | `sc_out<bool>` | 清除未完成的分支 |
| `reset` | `sc_out<bool>` | 重置信號 |

## 行為邏輯

Fetch 使用 `SC_CTHREAD` (clocked thread)，在每個時脈正緣觸發。

### 開機流程

1. 發出 `reset=true`，啟用記憶體讀取 (`ram_cs=true, ram_we=false`)
2. 等待 `memory_latency` 個週期讓資料出現
3. 等待 `bios_valid` 或 `icache_valid` 變為 true
4. 讀取第一筆指令，送出到 Decode
5. 當 PC 到達 5 時，解除 reset（BIOS 階段結束，切換到 ICache）

### 主迴圈

```
while (true):
    if 中斷發生:
        PC = 中斷向量位址
        讀取中斷處理程式的第一條指令
        送出中斷確認
    else if 分支有效:
        PC = 分支目標位址
        從新位址讀取指令
        設定 branch_clear
    else:
        從目前 PC 讀取指令
        PC++
```

### 關鍵觀察

- **Stall 機制**：當 `stall_fetch` 為 true 時，讀回的資料被丟棄（設為 0），類似於軟體中 backpressure 機制讓 producer 暫停。
- **記憶體延遲模擬**：`wait(memory_latency)` 模擬真實記憶體的存取延遲，這在軟體模擬中等同於 `sleep()` 或 `await`。
- **中斷優先於分支**：中斷檢查在分支檢查之前，確保硬體中斷能及時處理。

## SystemC 重點

- 使用 `SC_CTHREAD` 加上 `CLK.pos()` 表示此行為在每個時脈上升緣驅動。
- `wait()` 暫停執行直到下一個時脈邊緣，類似 `await nextTick()`。
- `do { wait(); } while (condition)` 是一個常見的「等待條件成立」模式，類似 busy-wait 但不消耗模擬時間以外的資源。
