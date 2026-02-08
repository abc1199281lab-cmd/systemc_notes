# 自頂向下 (Top-Down) 分析計劃

> **目標**: 將詳細的檔案級別分析綜合為高層次的架構理解，重點關注關係和依賴項。

## 執行策略
- **方法**: 將自底向上 (Bottom-Up) 分析的結果總結為具有凝聚力的主題文檔。
- **命名規範**: `doc/topdown/{topic}.md`

---

## 1. 核心架構 (大腦)
### 1.1 模擬引擎
- **檔案**: `doc/topdown/simulation_engine.md`
- **重點**: `sc_main`, `sc_simcontext` 和 `sc_time` 如何協同運作來驅動模擬。
- **關鍵檔案**: `sysc/kernel/sc_simcontext.cpp`, `sysc/kernel/sc_main.cpp`

### 1.2 行程排程與並行性
- **檔案**: `doc/topdown/scheduling.md`
- **重點**: Method (方法行程), Thread (執行緒行程) 和 CThread (時脈執行緒) 之間的區別，以及排程器如何挑選它們。
- **關鍵檔案**: `sysc/kernel/sc_process.cpp`, `sysc/kernel/sc_runnable.cpp`

### 1.3 事件系統
- **檔案**: `doc/topdown/events.md`
- **重點**: 事件如何觸發行程，以及 delta cycle (增量週期) 機制如何運作。
- **關鍵檔案**: `sysc/kernel/sc_event.cpp`

---

## 2. 結構階層 (身體)
### 2.1 模組與物件
- **檔案**: `doc/topdown/hierarchy.md`
- **重點**: 模組如何在物件階層中進行嵌套、命名和管理。
- **關鍵檔案**: `sysc/kernel/sc_module.cpp`, `sysc/kernel/sc_object_manager.cpp`

---

## 3. 通訊 (神經系統)
### 3.1 訊號與連接埠
- **檔案**: `doc/topdown/communication.md`
- **重點**: Interface-Port-Channel (介面-連接埠-通道) 模式。資料如何在模組之間移動。
- **關鍵檔案**: `sysc/communication/sc_signal.cpp`, `sysc/communication/sc_port.cpp`

---

## 4. 硬體資料型別 (血液)
### 4.1 資料表示
- **檔案**: `doc/topdown/datatypes.md`
- **重點**: 硬體特定型別 (位元向量、定點數) 如何在 C++ 中進行模擬。
- **關鍵檔案**: `sysc/datatypes/*`

---

## 5. 審閱與整合
- [ ] 將自底向上的分析結果綜合到這些自頂向下的文檔中。
- [ ] 確保每個自頂向下的文檔中都包含 "硬體 RTL 映射 (Hardware RTL Mapping)" 章節。
