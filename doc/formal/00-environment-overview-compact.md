# SystemC Phase 0: 環境準備與資源盤點

> **建立時間**: 2026-02-08  
> **SystemC 版本**: 3.0.2  
> **標準**: IEEE Std. 1666-2023

---

## 1. SystemC 簡介

SystemC 是 **硬體/軟體協同設計語言**，以 ANSI C++ 類別庫形式實現：

| 特性 | 說明 |
|-----|------|
| **C++ 類別庫** | 不是獨立語言，而是 C++ 的擴充 |
| **硬體模擬** | 支援 RTL 級、TLM 級等多層次建模 |
| **IEEE 標準** | IEEE Std. 1666-2023 正式標準 |
| **跨平台** | Linux/macOS/Windows 支援 |

### 1.1 主要用途

- 系統級建模 (System-level modeling)
- 架構探索 (Architectural exploration)
- 性能建模 (Performance modeling)
- 軟體開發 (Software development)
- 功能驗證 (Functional verification)
- 高階合成 (High-level synthesis)

---

## 2. 專案結構總覽

### 2.1 頂層目錄

```
ref/systemc/
├── src/                     # 原始碼（核心學習區）
│   ├── sysc/               # SystemC 核心實作
│   │   ├── kernel/         # 模擬核心（73 檔案）⭐
│   │   ├── communication/  # 通訊機制（42 檔案）⭐
│   │   ├── datatypes/      # 資料型別
│   │   │   ├── bit/        # 位元向量
│   │   │   ├── fx/         # 定點數
│   │   │   └── int/        # 整數型別
│   │   ├── tracing/        # 波形追蹤
│   │   └── utils/          # 工具函式
│   ├── tlm_core/           # TLM 核心
│   └── tlm_utils/          # TLM 工具
├── examples/               # 範例程式
│   ├── sysc/              # SystemC 範例
│   └── tlm/               # TLM 範例
├── docs/                   # 官方文件
├── tests/                  # 回歸測試
├── CMakeLists.txt          # CMake 配置
├── INSTALL.md              # 安裝說明
└── RELEASENOTES.md         # 版本說明
```

### 2.2 原始碼核心目錄 (src/sysc/)

| 目錄 | 檔案數 | 重要性 | 說明 |
|-----|-------|--------|------|
| `kernel/` | ~73 | ⭐⭐⭐ | 模擬核心引擎 |
| `communication/` | ~42 | ⭐⭐⭐ | 訊號、通道、埠 |
| `datatypes/bit/` | ~15 | ⭐⭐ | 位元/邏輯向量 |
| `datatypes/int/` | ~20 | ⭐⭐ | 整數型別 |
| `datatypes/fx/` | ~25 | ⭐ | 定點數 |
| `tracing/` | ~12 | ⭐⭐ | VCD 波形輸出 |
| `utils/` | ~8 | ⭐ | 工具函式 |

---

## 3. 編譯與安裝

### 3.1 需求

| 項目 | 最低版本 | 說明 |
|-----|---------|------|
| C++ 標準 | C++17 | IEEE 1666-2023 強制要求 |
| CMake | 3.5+ | 建議使用 |
| GCC | 9.3+ | Linux |
| Clang | 13.0+ | macOS/Linux |
| MSVC | 2019+ | Windows |

### 3.2 建置步驟

```bash
cd ref/systemc
mkdir build && cd build
cmake ..
make
make check        # 執行測試
sudo make install # 安裝到 /opt/systemc/
```

### 3.3 重要 CMake 選項

| 選項 | 預設 | 說明 |
|-----|------|------|
| `CMAKE_BUILD_TYPE` | Release | Debug/Release |
| `BUILD_SHARED_LIBS` | ON | 共享函式庫 |
| `ENABLE_EXAMPLES` | ON | 編譯範例 |
| `ENABLE_PTHREADS` | OFF | 使用 POSIX threads |
| `CMAKE_INSTALL_PREFIX` | /opt/systemc/ | 安裝路徑 |

---

## 4. 版本資訊 (3.0.2)

### 4.1 重要特性

- ✅ **完整實現 IEEE 1666-2023**
- ✅ **C++17 基線**（使用 `std::string_view`）
- ✅ **參考實作**（Accellera 官方）

### 4.2 已知問題

1. **Fixed-point** 在 MSVC x64 Release 模式有問題
2. **Small sc_time** 會被截斷為 SC_ZERO_TIME（無警告）
3. **Bit-wise 運算** 結果大小使用左運算元
4. **sc_dt::(u)int64** 與 std::(u)int64_t 定義可能不同

---

## 5. TLM 簡介

**Transaction Level Modeling (TLM)** 是 SystemC 的擴充：

| 層級 | 時間精度 | 用途 |
|-----|---------|------|
| **Loosely-timed (LT)** | 近似 | 軟體開發、架構分析 |
| **Approximately-timed (AT)** | 週期近似 | 性能建模 |

- 比 RTL **更快**（無時脈週期精確要求）
- 適合 **早期架構探索**

---

## 6. 學習筆記結構

```
doc/
├── draft/                          # 草稿（自由修改）
│   ├── 2025-02-08-00-environment-overview.md  # 本文件
│   ├── 2025-02-08-01-systemc-architecture.md  # Phase 1
│   ├── 2025-02-08-02-kernel-simcontext.md     # Phase 2
│   ├── ...
│   └── 2025-02-08-99-rtl-mapping.md           # RTL 對照表
│
└── formal/                         # 正式文件（審核後）
    └── ...
```

---

## 7. 參考資源

| 資源 | 連結 |
|-----|------|
| IEEE 標準 | https://ieeexplore.ieee.org/document/10246125 |
| Accellera | https://systemc.org/ |
| 討論區 | https://forums.accellera.org/forum/9-systemc/ |
| GitHub | https://github.com/accellera-official/systemc |

---

## 8. 進度追蹤

| Phase | 主題 | 狀態 | 日期 |
|:---:|:---|:---:|:---:|
| 0 | 環境準備 | ✅ | 2025-02-08 |
| 1 | 架構總覽 | ⏳ | - |
| 2 | Kernel 分析 | ⏳ | - |
| 3 | Communication 分析 | ⏳ | - |
| 4 | Datatypes 分析 | ⏳ | - |
| 5 | Tracing/Utils 分析 | ⏳ | - |
| 6 | TLM 分析 | ⏳ | - |
| 7 | 範例分析 | ⏳ | - |
| 8 | 整合總結 | ⏳ | - |

---

**下一步**: Phase 1 - Top-Down 架構總覽
