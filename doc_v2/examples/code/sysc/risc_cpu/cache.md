# Cache -- 指令快取與資料快取

## 軟體類比

Cache 在硬體中的角色就像軟體架構中的快取層（Redis、Memcached）。核心概念完全一樣：

```
CPU <-> L1 Cache <-> 主記憶體
App <-> Redis    <-> PostgreSQL
```

存取 Cache 比存取主記憶體快非常多。在這個範例中，ICache (指令快取) 和 DCache (資料快取) 是分開的，這種架構稱為 **Harvard Architecture** -- 就像你的應用程式分別用不同的快取存放程式碼和資料。

## 原始檔案

- `icache.h` / `icache.cpp` -- 指令快取
- `dcache.h` / `dcache.cpp` -- 資料快取

---

## ICache -- 指令快取

### 模組介面

| 方向 | 信號名稱 | 類型 | 說明 |
|------|-----------|------|------|
| 輸入 | `datain` | `sc_in<unsigned>` | 寫入資料 (用於 self-modifying code) |
| 輸入 | `cs` | `sc_in<bool>` | Chip Select（啟用） |
| 輸入 | `we` | `sc_in<bool>` | Write Enable |
| 輸入 | `addr` | `sc_in<unsigned>` | 存取位址 |
| 輸入 | `ld_valid` | `sc_in<bool>` | Process ID 載入有效 |
| 輸入 | `ld_data` | `sc_in<signed>` | Process ID 值 |
| 輸出 | `dataout` | `sc_out<unsigned>` | 讀取的指令資料 |
| 輸出 | `icache_valid` | `sc_out<bool>` | 輸出有效 |
| 輸出 | `stall_fetch` | `sc_out<bool>` | 暫停 Fetch |

### 內部結構

```cpp
unsigned *icmemory;       // 指令資料陣列 (最多 500 筆)
unsigned *ictagmemory;    // Tag 記憶體 (記錄哪些位址在快取中)
signed int pid;           // 當前 Process ID
```

- 初始化時從 `icache.img` 讀取指令內容
- 未初始化的位置填入 `0xeeeeeeee`（方便除錯辨識）

### 行為邏輯

```
while true:
    等待 cs == true
    if 位址 >= BOOT_LENGTH (5):   # 開機後才啟用 ICache
        if ld_valid:
            更新 Process ID
        if we == true:
            寫入 icmemory[address]    # Self-Modifying Code (SMC)
        else:
            讀取 icmemory[address]
            設定 icache_valid = true
    # 位址 < 5 的請求由 BIOS 處理
```

### 重要設計

- **BOOT_LENGTH = 5**：前 5 筆指令由 BIOS 提供，ICache 只處理位址 >= 5 的請求。
- **邊界檢查**：位址超過 `MAX_CODE_LENGTH` (500) 時輸出 `0xFFFFFFFF` 並印出警告。
- **Process ID 支援**：透過 `ld_valid` 和 `ld_data` 載入 PID，支援多程序環境下的快取切換。

---

## DCache -- 資料快取

### 模組介面

| 方向 | 信號名稱 | 類型 | 說明 |
|------|-----------|------|------|
| 輸入 | `datain` | `sc_in<signed>` | 寫入資料 |
| 輸入 | `statein` | `sc_in<unsigned>` | MESI 狀態位 |
| 輸入 | `cs` | `sc_in<bool>` | Chip Select |
| 輸入 | `we` | `sc_in<bool>` | Write Enable |
| 輸入 | `addr` | `sc_in<unsigned>` | 存取位址 |
| 輸入 | `dest` | `sc_in<unsigned>` | 目標暫存器 |
| 輸出 | `dataout` | `sc_out<signed>` | 讀取資料 |
| 輸出 | `destout` | `sc_out<unsigned>` | 目標暫存器（傳遞） |
| 輸出 | `out_valid` | `sc_out<bool>` | 輸出有效 |
| 輸出 | `stateout` | `sc_out<unsigned>` | MESI 狀態輸出 |

### 內部結構

```cpp
unsigned *dmemory;       // 資料陣列 (4000 筆)
unsigned *dsmemory;      // MESI 狀態陣列
unsigned *dtagmemory;    // Tag 記憶體
```

- 初始化時從 `dcache.img` 讀取資料
- 未初始化的位置填入 `0xdeadbeef`（這是一個經典的 debug magic number）

### MESI 協定

DCache 支援 MESI 快取一致性協定的狀態標記：

| 狀態值 | 代號 | 意義 | 軟體類比 |
|--------|------|------|----------|
| 3 | M (Modified) | 已修改，與主記憶體不一致 | 本地有未 push 的修改 |
| 2 | E (Exclusive) | 獨占，只有此快取有副本 | 本地 branch 未被 fork |
| 1 | S (Shared) | 共享，多個快取有副本 | 多人同時讀取同一份資料 |
| 0 | I (Invalid) | 無效 | Cache miss / 過期資料 |

在多核系統中，MESI 確保所有 CPU 看到一致的記憶體狀態，類似分散式系統中的 cache invalidation 策略。

### 行為邏輯

- **寫入**：將資料、狀態、tag 同時寫入對應位址
- **讀取**：輸出資料和 MESI 狀態，設定 `out_valid=true`，一個週期後清除

## 兩者比較

| 特性 | ICache | DCache |
|------|--------|--------|
| 初始化來源 | `icache.img` | `dcache.img` |
| 容量 | 500 entries | 4000 entries |
| 預設填充值 | `0xeeeeeeee` | `0xdeadbeef` |
| 寫入支援 | 有 (SMC) | 有 |
| MESI 支援 | 無 | 有 |
| Process ID | 有 | 無 |

## SystemC 重點

- 兩者都使用 `SC_CTHREAD`，在時脈正緣驅動。
- `do { wait(); } while (!(cs == true))` 是標準的「等待被啟用」模式。
- 快取記憶體用 C++ 動態陣列 (`new unsigned[N]`) 實作，而非 `sc_signal` -- 因為這是模組的內部狀態，不需要跨模組通訊。
