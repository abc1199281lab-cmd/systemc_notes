# 自底向上 (Bottom-Up) 分析計劃

> **目標**: 分析 `ref/systemc/src` 中的每個類別/函式，以了解設計與實作細節。

## 執行策略
- **順序**: 核心內核 (Core Kernel) -> 通訊 (Communication) -> 資料型別 (Datatypes) -> 工具類 (Utils) -> 追蹤 (Tracing) -> TLM
- **批次大小**: 每個批次 3 個檔案
- **命名規範**: `ref/systemc/src/sysc/kernel/sc_event.cpp` -> `doc/code/sysc/kernel/sc_event.md` (保留目錄結構)

---

## 1. SystemC 核心內核 (高優先權)
模擬引擎的核心。了解這些是了解其他所有內容的前提。

### 1.1 模擬控制與上下文
- [ ] `ref/systemc/src/sysc/kernel/sc_simcontext.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_main.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_main_main.cpp`

### 1.2 行程管理 (Process Management)
- [ ] `ref/systemc/src/sysc/kernel/sc_process.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_method_process.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_thread_process.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_cthread_process.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_spawn_options.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_join.cpp`

### 1.3 事件與時間
- [ ] `ref/systemc/src/sysc/kernel/sc_event.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_time.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_wait.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_wait_cthread.cpp`

### 1.4 模組與物件階層
- [ ] `ref/systemc/src/sysc/kernel/sc_object.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_object_manager.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_module.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_module_name.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_module_registry.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_name_gen.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_attribute.cpp`

### 1.5 協程 (平台特定)
- [ ] `ref/systemc/src/sysc/kernel/sc_cor_pthread.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_cor_qt.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_cor_std_thread.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_cor_fiber.cpp`

### 1.6 其他內核相關
- [ ] `ref/systemc/src/sysc/kernel/sc_reset.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_sensitive.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_stage_callback_registry.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_except.cpp`
- [ ] `ref/systemc/src/sysc/kernel/sc_ver.cpp`

---

## 2. 通訊 (中優先權)
連接模組的通道、連接埠和訊號。

### 2.1 基礎介面與連接埠
- [ ] `ref/systemc/src/sysc/communication/sc_interface.cpp`
- [ ] `ref/systemc/src/sysc/communication/sc_port.cpp`
- [ ] `ref/systemc/src/sysc/communication/sc_export.cpp`

### 2.2 訊號 (Signal)
- [ ] `ref/systemc/src/sysc/communication/sc_signal.cpp`
- [ ] `ref/systemc/src/sysc/communication/sc_signal_ports.cpp`
- [ ] `ref/systemc/src/sysc/communication/sc_signal_resolved.cpp`
- [ ] `ref/systemc/src/sysc/communication/sc_signal_resolved_ports.cpp`

### 2.3 其他通道
- [ ] `ref/systemc/src/sysc/communication/sc_prim_channel.cpp`
- [ ] `ref/systemc/src/sysc/communication/sc_clock.cpp`
- [ ] `ref/systemc/src/sysc/communication/sc_mutex.cpp`
- [ ] `ref/systemc/src/sysc/communication/sc_semaphore.cpp`
- [ ] `ref/systemc/src/sysc/communication/sc_event_queue.cpp`
- [ ] `ref/systemc/src/sysc/communication/sc_event_finder.cpp`

---

## 3. 資料型別 (低優先權 - 主要作為參考)
硬體資料型別的實作。

### 3.1 位元與邏輯 (Bit & Logic)
- [ ] `ref/systemc/src/sysc/datatypes/bit/sc_bit.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/bit/sc_logic.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/bit/sc_bv_base.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/bit/sc_lv_base.cpp`

### 3.2 整數 (Integer)
- [ ] `ref/systemc/src/sysc/datatypes/int/sc_int_base.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/int/sc_uint_base.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/int/sc_signed.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/int/sc_unsigned.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/int/sc_int64_io.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/int/sc_int64_mask.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/int/sc_int32_mask.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/int/sc_length_param.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/int/sc_nbutils.cpp`

### 3.3 定點數 (Fixed Point)
- [ ] `ref/systemc/src/sysc/datatypes/fx/sc_fxnum.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/fx/sc_fxval.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/fx/scfx_rep.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/fx/scfx_mant.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/fx/scfx_utils.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/fx/scfx_pow10.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/fx/sc_fxdefs.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/fx/sc_fxcast_switch.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/fx/sc_fxtype_params.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/fx/sc_fxnum_observer.cpp`
- [ ] `ref/systemc/src/sysc/datatypes/fx/sc_fxval_observer.cpp`

### 3.4 其他型別
- [ ] `ref/systemc/src/sysc/datatypes/misc/sc_value_base.cpp`

---

## 4. 追蹤 (Tracing) (低優先權)
波形追蹤支援。
- [ ] `ref/systemc/src/sysc/tracing/sc_trace.cpp`
- [ ] `ref/systemc/src/sysc/tracing/sc_trace_file_base.cpp`
- [ ] `ref/systemc/src/sysc/tracing/sc_vcd_trace.cpp`
- [ ] `ref/systemc/src/sysc/tracing/sc_wif_trace.cpp`

---

## 5. 工具類 (Utils) (低優先權)
- [ ] `ref/systemc/src/sysc/utils/sc_report.cpp`
- [ ] `ref/systemc/src/sysc/utils/sc_report_handler.cpp`
- [ ] `ref/systemc/src/sysc/utils/sc_vector.cpp`
- [ ] `ref/systemc/src/sysc/utils/sc_list.cpp`
- [ ] `ref/systemc/src/sysc/utils/sc_mempool.cpp`
- [ ] `ref/systemc/src/sysc/utils/sc_pq.cpp` (優先權佇列)
- [ ] `ref/systemc/src/sysc/utils/sc_hash.cpp`
- [ ] `ref/systemc/src/sysc/utils/sc_stop_here.cpp`
- [ ] `ref/systemc/src/sysc/utils/sc_utils_ids.cpp`

---

## 6. TLM 核心與工具 (中優先權)
事務級建模 (Transaction Level Modeling)。
- [ ] `ref/systemc/src/tlm_core/tlm_2/tlm_generic_payload/tlm_gp.cpp`
- [ ] `ref/systemc/src/tlm_core/tlm_2/tlm_generic_payload/tlm_phase.cpp`
- [ ] `ref/systemc/src/tlm_core/tlm_2/tlm_quantum/tlm_global_quantum.cpp`
- [ ] `ref/systemc/src/tlm_utils/convenience_socket_bases.cpp`
- [ ] `ref/systemc/src/tlm_utils/instance_specific_extensions.cpp`
