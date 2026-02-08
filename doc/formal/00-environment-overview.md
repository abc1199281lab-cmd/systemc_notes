# SystemC 環境準備與資源盤點

> **建立日期**: 2026-02-08  
> **版本**: SystemC 3.0.2  
> **標準**: IEEE Std. 1666-2023

---

## 1. SystemC 是什麼？

SystemC 是一個基於 C++ 的系統級設計與驗證語言，主要用途包括：

- **系統級建模** (System-level modeling)
- **架構探索** (Architectural exploration)
- **性能建模** (Performance modeling)
- **軟體開發** (Software development)
- **功能驗證** (Functional verification)
- **高階合成** (High-level synthesis)
- **抽象類比/混合訊號建模** (Abstract analog/mixed-signal modeling)

### 1.1 核心特點

| 特點 | 說明 |
|-----|------|
| **C++ 類別庫** | 以 ANSI C++ 類別庫形式實現，非獨立語言 |
| **硬體 + 軟體** | 跨越硬體與軟體的系統設計語言 |
| **IEEE 標準** | 正式標準為 IEEE Std. 1666-2023 |
| **Accellera 維護** | 由 Accellera Systems Initiative 開發 |

---

## 2. 專案結構總覽

### 2.1 頂層目錄

```
ref/systemc/
├── .git/                  # Git 版本控制
├── .github/               # GitHub 相關配置
├── cmake/                 # CMake 配置檔
├── config/                # 編譯配置
├── docker/                # Docker 配置
├── docs/                  # 官方文件
├── examples/              # 範例程式
│   ├── sysc/             # SystemC 核心範例
│   └── tlm/              # TLM 範例
├── msvc16/                # Visual Studio 2019 專案檔
├── src/                   # 原始碼（核心！）
│   ├── sysc/             # SystemC 核心實作
│   ├── tlm_core/         # TLM 核心
│   └── tlm_utils/        # TLM 工具
├── tests/                 # 回歸測試
├── CMakeLists.txt         # CMake 主配置
├── configure.ac           # Autotools 配置（legacy）
├── Makefile.am            # Automake 配置（legacy）
├── INSTALL.md             # 安裝說明
├── LICENSE                # 授權條款
├── NOTICE                 # 版權聲明
├── README.md              # 專案說明
└── RELEASENOTES.md        # 版本說明
```

---

## 3. 原始碼結構（src/）

這是理解 SystemC 的核心區域：

```
src/
├── sysc/                          # SystemC 核心實作
│   ├── kernel/                    # 模擬核心 (73 個檔案)
│   │   ├── sc_simcontext.cpp/h   # 模擬上下文
│   │   ├── sc_process*.cpp/h     # 各類 Process
│   │   ├── sc_event.cpp/h        # 事件機制
│   │   ├── sc_module*.cpp/h      # 模組機制
│   │   ├── sc_time.cpp/h         # 時間機制
│   │   └── sc_cor*.cpp/h         # Coroutine 實作
│   │
│   ├── communication/             # 通訊機制 (42 個檔案)
│   │   ├── sc_interface.cpp/h    # 介面基礎
│   │   ├── sc_port.cpp/h         # 埠機制
│   │   ├── sc_signal*.cpp/h      # 訊號機制
│   │   ├── sc_fifo*.h            # FIFO 通道
│   │   ├── sc_mutex.cpp/h        # 互斥鎖
│   │   └── sc_clock.cpp/h        # 時脈機制
│   │
│   ├── datatypes/                 # 資料型別
│   │   ├── bit/                   # 位元向量 (sc_bit, sc_bv, sc_logic, sc_lv)
│   │   ├── fx/                    # 定點數
│   │   ├── int/                   # 整數型別 (sc_int, sc_bigint)
│   │   └── misc/                  # 其他輔助型別
│   │
│   ├── tracing/                   # 波形追蹤 (VCD 輸出)
│   ├── utils/                     # 工具函式
│   └── packages/                  # 套件
│
├── tlm_core/                      # TLM 核心
│   └── tlm_2/                     # TLM 2.0
│       ├── tlm_generic_payload/   # 通用載入
│       ├── tlm_mm/                # 記憶體管理
│       ├── tlm_phase/             # 傳輸階段
│       └── tlm_sockets/           # Socket 介面
│
└── tlm_utils/                     # TLM 工具
    ├── instance_specific_extensions.hpp
    ├── multi_passthrough_initiator_socket.h
    ├── multi_passthrough_target_socket.h
    ├── passthrough_target_socket.h
    ├── peq_with_cb_and_phase.h      # Payload Event Queue
    ├── simple_initiator_socket.h
    └── simple_target_socket.h
```

---

## 4. 編譯需求

### 4.1 支援平台

| 作業系統 | 架構 | 編譯器 |
|---------|------|--------|
| Linux | x86_64, x86 (32-bit), arm64 | GCC 9.3+, Clang 13+ |
| macOS | Apple Silicon, x86_64 | AppleClang 14+ |
| Windows | x86_64 | MinGW/MSYS GCC, MSVC 2019+ |

### 4.2 必要工具

1. **CMake** (>= 3.5) - 建議使用 CMake 建置流程
2. **C++17 編譯器** - IEEE Std. 1666-2023 要求 C++17
3. **GNU Make** 或 **Ninja**

### 4.3 建置步驟

```bash
# 1. 建立 build 目錄
cd ref/systemc
mkdir build && cd build

# 2. 配置 (CMake GUI 或命令列)
ccmake ..
# 或
cmake ..

# 3. 編譯
make

# 4. 測試
make check

# 5. 安裝
sudo make install  # 預設安裝到 /opt/systemc/
```

### 4.4 重要 CMake 選項

| 選項 | 預設 | 說明 |
|-----|------|------|
| `BUILD_SHARED_LIBS` | ON (非 Windows) | 建立共享函式庫 |
| `CMAKE_BUILD_TYPE` | Release | 建置類型 |
| `CMAKE_CXX_STANDARD` | 17 | C++ 標準 |
| `CMAKE_INSTALL_PREFIX` | /opt/systemc/ | 安裝路徑 |
| `ENABLE_EXAMPLES` | ON | 編譯範例 |
| `ENABLE_REGRESSION` | OFF | 編譯回歸測試 |
| `ENABLE_PTHREADS` | OFF | 使用 POSIX threads |

---

## 5. 版本資訊 (SystemC 3.0.2)

### 5.1 重要特性

- **實現 IEEE Std. 1666-2023** 標準
- **C++17 基線** - 使用 `std::string_view` 等新特性
- **參考實作** - 由 Accellera Systems Initiative 提供

### 5.2 已知問題

1. **Fixed-point** 在 MSVC 2017+ x64 Release 模式有問題
2. **Small sc_time** 會被截斷為 SC_ZERO_TIME（無警告）
3. **Bit-wise 運算** 結果大小使用左運算元而非較大者
4. **sc_dt::(u)int64** 與 std::(u)int64_t 在某些平台定義不同

---

## 6. 官方文件資源

### 6.1 主要參考

| 資源 | 位置 | 說明 |
|-----|------|------|
| **IEEE 標準** | IEEE Std. 1666-2023 | 正式語言參考手冊 |
| **安裝說明** | `ref/systemc/INSTALL.md` | 編譯與安裝指南 |
| **版本說明** | `ref/systemc/RELEASENOTES.md` | 版本特性與已知問題 |
| **開發文件** | `ref/systemc/docs/` | 額外開發文件 |

### 6.2 線上資源

- **Accellera SystemC**: https://systemc.org/
- **SystemC 論壇**: https://forums.accellera.org/forum/9-systemc/
- **GitHub 倉庫**: https://github.com/accellera-official/systemc
- **IEEE 標準**: https://ieeexplore.ieee.org/document/10246125

---

## 7. TLM (Transaction Level Modeling) 基礎

### 7.1 什麼是 TLM？

TLM 是 SystemC 的擴充，提供**交易級**抽象層次：

- **比 RTL 更高層級** - 不需要時脈週期精確
- **快速模擬** - 適合早期架構探索
- **介面標準化** - 通用載入 (generic payload)

### 7.2 TLM 抽象層次

| 層級 | 時間精度 | 用途 |
|-----|---------|------|
| **Loosely-timed (LT)** | 時間近似 | 軟體開發、架構分析 |
| **Approximately-timed (AT)** | 週期近似 | 性能建模、硬體/軟體協同設計 |

---

## 8. 學習筆記結構

### 8.1 筆記目錄規劃

```
doc/
├── formal/                         # 學習筆記
│   ├── 00-environment-overview.md # 本文件
│   ├── 01-systemc-architecture.md # Phase 1: 架構總覽
│   ├── 02-kernel-simcontext.md    # Phase 2: 模擬核心
│   ├── 02-kernel-process.md
│   ├── 02-kernel-event.md
│   ├── 02-kernel-module.md
│   ├── 03-communication-signal.md # Phase 3: 通訊機制
│   ├── 03-communication-channel.md
│   ├── 04-datatypes-bit.md        # Phase 4: 資料型別
│   ├── ...
│   └── 99-rtl-mapping.md          # RTL 對照表
│
└── formal/                        # 正式文件
    └── (整理後的文件)
```

### 8.2 文件格式

每個分析文件將包含：
- **模組概述** - 職責與架構角色
- **檔案結構** - 相關檔案清單
- **核心類別** - 類別設計與方法
- **執行流程** - 運作機制說明
- **RTL 對照** - 對應硬體概念
- **使用範例** - 程式碼範例

---

## 9. 學習目標總結

### 9.1 Top-Down 目標

1. 理解 SystemC 整體架構設計
2. 掌握 Module/Process/Event/Channel 核心概念
3. 理解模擬執行模型 (delta cycle, time advance)

### 9.2 Bottom-Up 目標

1. 深入分析每個核心檔案的 class/function 設計
2. 理解 coroutine/context switch 實作機制
3. 掌握 signal update/request-update pattern

### 9.3 RTL 知識補充

1. 建立 SystemC 概念與 Verilog/RTL 的對應關係
2. 理解硬體設計原理 (clock edge, combinational/sequential logic)

---

## 10. 下一步：Phase 1

**Phase 1: Top-Down 架構總覽**

- [ ] 繪製 SystemC 整體架構 Mermaid 圖
- [ ] 說明 SystemC 與傳統軟體框架的差異
- [ ] 解釋為什麼需要這樣的設計（硬體模擬需求）
- [ ] 核心概念說明 (Module, Process, Event, Channel, Simulation)

**輸出物**: `doc/draft/01-systemc-architecture-overview.md`

---

## 附錄：檔案大小統計

快速了解核心檔案規模：

| 目錄 | 檔案數 | 說明 |
|-----|-------|------|
| `sysc/kernel/` | ~73 個 | 模擬核心（最大） |
| `sysc/communication/` | ~42 個 | 通訊機制 |
| `sysc/datatypes/` | ~40 個 | 資料型別 |
| `sysc/tracing/` | ~12 個 | 波形追蹤 |
| `sysc/utils/` | ~8 個 | 工具函式 |
| `tlm_core/` | ~15 個 | TLM 核心 |
| `tlm_utils/` | ~8 個 | TLM 工具 |

---

> **備註**: 本文件為學習計畫的起點，後續將逐步深入分析每個模組的設計與實作。
