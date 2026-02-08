# SystemC Notes Granular Plan

## Objective
提供 SystemC 核心機制（尤其是模擬核心與通訊模型）的深度解析，確保每個核心概念都有獨立、詳盡的文件，而不僅僅是高階概述。

## Documentation Structure (Granular)

### Phase 1: 架構與環境 (Architecture & Env)
- [x] `00-environment-overview.md`: 專案結構與環境說明。
- [x] `01-systemc-architecture.md`: SystemC 層級結構（Language, Channels, Core, C++）。

### Phase 2: 模擬核心深度剖析 (Kernel & Simulation Context)
- [x] `02-kernel-simcontext.md`: `sc_simcontext` 的職責、動態機制與全域狀態管理。
- [x] `02-kernel-scheduler.md`: 模擬排程器（Scheduler）的五個階段（Initialization, Evaluation, Update, Delta Notification, Timed Notification）。
- [x] `02-kernel-process.md`: `SC_METHOD`, `SC_THREAD`, `SC_CTHREAD` 的底層實作與區別。
- [x] `02-kernel-event.md`: `sc_event` 隊列、動態敏感度（Dynamic Sensitivity）與事件觸發路徑。
- [x] `02-kernel-module.md`: `sc_module` 的構造過程、`before_end_of_elaboration` 等生命週期回呼。

### Phase 3: 通訊機制與介面 (Communication & Interfaces)
- [x] `03-communication-interface.md`: `sc_interface` 與 `sc_port` 的抽象層次。
- [x] `03-communication-channel.md`: 階層式頻道（Hierarchical Channels）與基本頻道（Primitive Channels）的實作。
- [ ] `03-communication-signal.md`: `sc_signal` 的 `write()` 與 `update()` 分離機制（避免 Race Condition）。
- [ ] `03-communication-fifo.md`: `sc_fifo` 的同步阻塞機制分析。

### Phase 4: 資料型別與硬體映射 (Data Types & RTL)
- [ ] `04-datatypes-bit-int.md`: `sc_bit`, `sc_logic`, `sc_int`, `sc_bigint` 的位元寬度處理與效能考量。
- [ ] `04-datatypes-fixed.md`: 定點數（Fixed-point）運算與捨入模式。
- [ ] `99-rtl-mapping.md`: SystemC 語義與 Verilog/VHDL RTL 的對照分析（如 `always` block 映射）。

### Phase 5: TLM 2.0 深度分析
- [ ] `05-tlm-initiator-target.md`: Socket 通訊模型與介面。
- [ ] `05-tlm-payload.md`: 通用負載（Generic Payload）與延伸屬性（Extensions）。
- [ ] `05-tlm-blocking-nonblocking.md`: LT (Loose Timing) 與 AT (Approximately Timed) 模型的底層差異。

## Workflow
- **One Concept, One File**: 每個子文件專注於一個特定的類別或機制。
- **Source-Driven**: 必須引用 `ref/systemc` 中的原始碼作為佐證。
- **Commit Strategy**: 每完成一個子章節即提交至 `dev` 分支。
