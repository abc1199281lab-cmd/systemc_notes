# SystemC 學習計畫 (Study Plan)

> 本計畫旨在深入理解 SystemC 程式庫，採用 **Top-Down 架構設計** 與 **Bottom-Up 程式碼分析** 雙軌並進的方式，產出適合軟體背景使用者閱讀的技術文件。

---

## 目標 (Goal)

根據 [AGENTS.md](file:///Users/ai_lab/Developer/systemc_notes/AGENTS.md) 定義：
- 深入理解 `ref/systemc` 的架構設計（Top-Down）
- 理解每個檔案裡每個 class/functions 的設計與運作原理（Bottom-Up）
- 適時補充硬體 RTL 知識與設計原理

---

## 讀者對象 (Target Audience)

- 軟體背景的博士生
- 較少硬體 RTL 訓練經驗
- 喜歡簡單、有條理、連國中生都能懂的說明
- 需要在相關 code 補充硬體知識

---

## Phase 0：環境準備與資源盤點
**時間估計**: 1-2 小時

### 0.1 了解專案結構
- [ ] 閱讀 `ref/systemc/README.md` — 了解 SystemC 是什麼
- [ ] 閱讀 `ref/systemc/INSTALL.md` — 了解如何編譯與安裝
- [ ] 閱讀 `ref/systemc/RELEASENOTES.md` — 了解版本特性

### 0.2 官方文件與標準
- [ ] 了解 IEEE Std. 1666-2023 標準的定位
- [ ] 瀏覽 `ref/systemc/docs/` 目錄中的開發文件
- [ ] 了解 TLM (Transaction Level Modeling) 的基本概念

### 0.3 建立學習筆記結構
- [ ] 整理 `doc/draft/` 用於暫存筆記
- [ ] 確認 `doc/formal/` 用於正式文件

**輸出物**:
- `doc/draft/YYYY-MM-DD-HH-MM-SS-environment-overview.md`

---

## Phase 1：Top-Down 架構總覽
**時間估計**: 4-6 小時

### 1.1 SystemC 核心概念架構圖
- [ ] 繪製 SystemC 整體架構 Mermaid 圖
- [ ] 說明 SystemC 與傳統軟體框架的差異
- [ ] 解釋為什麼需要這樣的設計（硬體模擬需求）

### 1.2 目錄結構分析

```
ref/systemc/src/
├── sysc/                    # SystemC 核心實作
│   ├── kernel/              # 模擬核心（73 個檔案）
│   ├── communication/       # 通訊機制（42 個檔案）
│   ├── datatypes/           # 資料型別
│   │   ├── bit/             # 位元向量
│   │   ├── fx/              # 定點數
│   │   ├── int/             # 整數型別
│   │   └── misc/            # 其他
│   ├── tracing/             # 波形追蹤
│   ├── utils/               # 工具函式
│   └── packages/            # 套件
├── tlm_core/                # TLM 核心
├── tlm_utils/               # TLM 工具
└── (header files)           # 公開標頭檔
```

### 1.3 核心概念說明
- [ ] 說明 **Module** 的概念（對應硬體的「元件」）
- [ ] 說明 **Process** 的概念（對應硬體的「行為」）
- [ ] 說明 **Event** 與 **Channel** 的概念（對應硬體的「信號」）
- [ ] 說明 **Simulation** 的執行模型

**輸出物**:
- `doc/draft/YYYY-MM-DD-HH-MM-SS-systemc-architecture-overview.md`
- 包含 Mermaid 架構圖

---

## Phase 2：Kernel 模組深入分析
**時間估計**: 8-12 小時

### 2.1 Simulation Context (`sc_simcontext`)
**核心檔案**: `kernel/sc_simcontext.cpp` (73KB), `kernel/sc_simcontext.h`

- [ ] 理解模擬器的生命週期
- [ ] 理解 elaboration phase vs simulation phase
- [ ] 理解 delta cycle 與時間推進機制
- [ ] **RTL 知識補充**: 為什麼需要 delta cycle？

> **硬體背景說明**: Delta cycle 對應到硬體模擬中「同時發生」的事件處理方式

### 2.2 Process 機制
**核心檔案**:
- `kernel/sc_process.cpp` / `.h` — 基礎 process
- `kernel/sc_method_process.cpp` / `.h` — METHOD process
- `kernel/sc_thread_process.cpp` / `.h` — THREAD process
- `kernel/sc_cthread_process.cpp` / `.h` — CTHREAD process

- [ ] 理解三種 process 類型的差異
- [ ] 理解 process 的排程與執行
- [ ] 理解 sensitivity list 的實作
- [ ] **RTL 知識補充**: 對應 Verilog 的 `always @` 區塊

### 2.3 Event 機制
**核心檔案**: `kernel/sc_event.cpp` / `.h`

- [ ] 理解 event 的 notify 機制
- [ ] 理解 immediate / delta / timed notification
- [ ] 理解 event queue 與 event list
- [ ] **RTL 知識補充**: 對應硬體的 clock edge 與 signal change

### 2.4 Module 機制
**核心檔案**:
- `kernel/sc_module.cpp` / `.h`
- `kernel/sc_module_name.cpp` / `.h`
- `kernel/sc_object.cpp` / `.h`

- [ ] 理解 module hierarchy
- [ ] 理解 elaboration 過程
- [ ] 理解 name management
- [ ] **RTL 知識補充**: 對應 Verilog 的 module 階層

### 2.5 Time 機制
**核心檔案**: `kernel/sc_time.cpp` / `.h`

- [ ] 理解 time resolution
- [ ] 理解 time 的表示方式
- [ ] **RTL 知識補充**: 模擬時間 vs 真實時間

### 2.6 Coroutine 實作分析
**核心檔案**:
- `kernel/sc_cor_pthread.cpp` / `.h` — POSIX thread 版本
- `kernel/sc_cor_qt.cpp` / `.h` — QuickThreads 版本
- `kernel/sc_cor_fiber.cpp` / `.h` — Windows Fiber 版本
- `kernel/sc_cor_std_thread.cpp` / `.h` — C++ std::thread 版本

- [ ] 理解 process 的 context switch 實作方式
- [ ] 比較不同平台的 coroutine 實作

**輸出物**:
- `doc/draft/YYYY-MM-DD-HH-MM-SS-kernel-simcontext.md`
- `doc/draft/YYYY-MM-DD-HH-MM-SS-kernel-process.md`
- `doc/draft/YYYY-MM-DD-HH-MM-SS-kernel-event.md`
- `doc/draft/YYYY-MM-DD-HH-MM-SS-kernel-module.md`

---

## Phase 3：Communication 模組深入分析
**時間估計**: 6-8 小時

### 3.1 Interface 與 Port 基礎
**核心檔案**:
- `communication/sc_interface.cpp` / `.h`
- `communication/sc_port.cpp` / `.h`

- [ ] 理解 interface-based design
- [ ] 理解 port 與 interface 的綁定機制
- [ ] **RTL 知識補充**: 對應 Verilog 的 port declaration

### 3.2 Signal 機制
**核心檔案**:
- `communication/sc_signal.cpp` / `.h`
- `communication/sc_signal_ifs.h`
- `communication/sc_signal_ports.cpp` / `.h`

- [ ] 理解信號的讀寫機制
- [ ] 理解信號的 update 機制（request-update pattern）
- [ ] 理解不同的 writer policy
- [ ] **RTL 知識補充**: 對應 Verilog 的 wire/reg

### 3.3 Resolved Signal
**核心檔案**:
- `communication/sc_signal_resolved.cpp` / `.h`
- `communication/sc_signal_rv.h`

- [ ] 理解多驅動器信號的解析
- [ ] 理解四值邏輯 (0, 1, X, Z)
- [ ] **RTL 知識補充**: 對應硬體的 tri-state buffer

### 3.4 Channel 機制
**核心檔案**:
- `communication/sc_prim_channel.cpp` / `.h`
- `communication/sc_fifo.h`
- `communication/sc_fifo_ifs.h`
- `communication/sc_fifo_ports.h`

- [ ] 理解 primitive channel 的 request_update 機制
- [ ] 理解 FIFO channel 的實作
- [ ] **RTL 知識補充**: 對應硬體的 handshaking protocol

### 3.5 同步機制
**核心檔案**:
- `communication/sc_mutex.cpp` / `.h`
- `communication/sc_semaphore.cpp` / `.h`

- [ ] 理解互斥鎖的實作
- [ ] 理解信號量的實作
- [ ] **RTL 知識補充**: 硬體中的 arbiter 概念

### 3.6 Clock 機制
**核心檔案**: `communication/sc_clock.cpp` / `.h`

- [ ] 理解 clock 的產生機制
- [ ] 理解 clock 與 process 的關係
- [ ] **RTL 知識補充**: 對應硬體的 clock domain

**輸出物**:
- `doc/draft/YYYY-MM-DD-HH-MM-SS-communication-signal.md`
- `doc/draft/YYYY-MM-DD-HH-MM-SS-communication-channel.md`
- `doc/draft/YYYY-MM-DD-HH-MM-SS-communication-clock.md`

---

## Phase 4：Datatypes 模組深入分析
**時間估計**: 6-8 小時

### 4.1 Bit 資料型別
**核心目錄**: `datatypes/bit/`

- [ ] 理解 `sc_bit` 與 `sc_logic` 的差異
- [ ] 理解 `sc_bv` (bit vector) 的實作
- [ ] 理解 `sc_lv` (logic vector) 的實作
- [ ] **RTL 知識補充**: 對應 Verilog 的 `reg [n:0]`

### 4.2 Integer 資料型別
**核心目錄**: `datatypes/int/`

- [ ] 理解 `sc_int` / `sc_uint` 的實作
- [ ] 理解 `sc_bigint` / `sc_biguint` 的實作
- [ ] 理解有號/無號整數的差異
- [ ] **RTL 知識補充**: 硬體中的 signed/unsigned arithmetic

### 4.3 Fixed-Point 資料型別
**核心目錄**: `datatypes/fx/`

- [ ] 理解定點數的表示方式
- [ ] 理解 overflow 與 quantization 模式
- [ ] **RTL 知識補充**: DSP 應用中的定點數設計

### 4.4 其他工具型別
**核心目錄**: `datatypes/misc/`

- [ ] 理解其他輔助型別

**輸出物**:
- `doc/draft/YYYY-MM-DD-HH-MM-SS-datatypes-bit.md`
- `doc/draft/YYYY-MM-DD-HH-MM-SS-datatypes-int.md`
- `doc/draft/YYYY-MM-DD-HH-MM-SS-datatypes-fx.md`

---

## Phase 5：Tracing 與 Utils 模組分析
**時間估計**: 3-4 小時

### 5.1 Tracing 波形追蹤
**核心目錄**: `sysc/tracing/`

- [ ] 理解 VCD (Value Change Dump) 格式
- [ ] 理解 trace file 的產生機制
- [ ] **RTL 知識補充**: 波形檢視器 (waveform viewer) 的使用

### 5.2 Utils 工具函式
**核心目錄**: `sysc/utils/`

- [ ] 理解 report handler 機制
- [ ] 理解其他工具函式

**輸出物**:
- `doc/draft/YYYY-MM-DD-HH-MM-SS-tracing-overview.md`

---

## Phase 6：TLM (Transaction Level Modeling) 分析
**時間估計**: 6-8 小時

### 6.1 TLM 基礎概念
**核心目錄**: `tlm_core/`, `tlm_utils/`

- [ ] 理解 TLM 的抽象層次
- [ ] 理解 loosely-timed vs approximately-timed
- [ ] 理解 TLM generic payload

### 6.2 TLM 通訊機制
- [ ] 理解 initiator 與 target
- [ ] 理解 blocking vs non-blocking transport
- [ ] 理解 DMI (Direct Memory Interface)

### 6.3 TLM 與 RTL 的關係
- [ ] **RTL 知識補充**: TLM 如何對應到實際硬體匯流排

**輸出物**:
- `doc/draft/YYYY-MM-DD-HH-MM-SS-tlm-overview.md`

---

## Phase 7：範例程式分析
**時間估計**: 4-6 小時

### 7.1 基礎範例
**核心目錄**: `examples/sysc/`

- [ ] 分析簡單的 hello world 範例
- [ ] 分析 counter 範例
- [ ] 分析 producer-consumer 範例

### 7.2 進階範例
- [ ] 分析更複雜的系統範例
- [ ] 理解實際應用模式

### 7.3 TLM 範例
**核心目錄**: `examples/tlm/`

- [ ] 分析 TLM 範例程式

**輸出物**:
- `doc/draft/YYYY-MM-DD-HH-MM-SS-examples-analysis.md`

---

## Phase 8：整合與總結
**時間估計**: 4-6 小時

### 8.1 完整架構圖繪製
- [ ] 整合所有模組關係
- [ ] 繪製完整的 class diagram (Mermaid)
- [ ] 繪製完整的 sequence diagram (Mermaid)

### 8.2 正式文件產出
- [ ] 將草稿整理成正式文件
- [ ] 移動至 `doc/formal/`
- [ ] 建立目錄索引

### 8.3 RTL 對照表
- [ ] 整理 SystemC 概念與 RTL 的對照表
- [ ] 適合軟體背景讀者參考

**輸出物**:
- `doc/formal/YYYY-MM-DD-HH-MM-SS-systemc-complete-guide.md`
- `doc/formal/YYYY-MM-DD-HH-MM-SS-systemc-rtl-mapping.md`

---

## 附錄：優先順序建議

### 高優先（必讀）
1. **Phase 1**: 架構總覽
2. **Phase 2.1-2.4**: Kernel 核心 (simcontext, process, event, module)
3. **Phase 3.1-3.2**: Signal 與 Port 基礎

### 中優先（建議閱讀）
4. **Phase 3.3-3.6**: 其他 communication 機制
5. **Phase 4.1-4.2**: Bit 與 Integer 型別
6. **Phase 6**: TLM 概念

### 低優先（選讀）
7. **Phase 4.3**: Fixed-point
8. **Phase 5**: Tracing
9. **Phase 7**: 範例分析

---

## 進度追蹤

| Phase | 主題 | 狀態 | 開始日期 | 完成日期 |
|:---:|:---|:---:|:---:|:---:|
| 0 | 環境準備 | [ ] | - | - |
| 1 | 架構總覽 | [ ] | - | - |
| 2 | Kernel 分析 | [ ] | - | - |
| 3 | Communication 分析 | [ ] | - | - |
| 4 | Datatypes 分析 | [ ] | - | - |
| 5 | Tracing/Utils 分析 | [ ] | - | - |
| 6 | TLM 分析 | [ ] | - | - |
| 7 | 範例分析 | [ ] | - | - |
| 8 | 整合總結 | [ ] | - | - |

---

## 參考資源

- [IEEE Std. 1666-2023](https://ieeexplore.ieee.org/document/10246125) — SystemC 正式標準
- [Accellera SystemC](https://systemc.org/) — 官方社群網站
- [SystemC Forum](https://forums.accellera.org/forum/9-systemc/) — 社群討論區
- `ref/nanobot_notes/doc/code` — Bottom-up 分析參考
