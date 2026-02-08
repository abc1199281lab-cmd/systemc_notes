# TLM DMI 分析

> **檔案**: `ref/systemc/src/tlm_core/tlm_2/tlm_2_interfaces/tlm_dmi.h`

## 1. 概述
DMI (直接記憶體介面, Direct Memory Interface) 是 TLM 2.0 中的一種機制，允許發起者繞過傳輸呼叫 (`b_transport` 或 `nb_transport`)，直接存取目標的記憶體陣列。這能顯著提高記憶體密集型工作負載的模擬速度。

## 2. `tlm_dmi` 資料結構
`tlm_dmi` 類別充當 DMI 資訊的容器：
- **`m_dmi_ptr`**: 指向目標內部儲存空間的原始指標 (`unsigned char*`)。
- **地址範圍**: `m_dmi_start_address` 和 `m_dmi_end_address` 定義了此指標的有效區域 (通常相對於目標的地址空間)。
- **存取權限**: `m_dmi_access` 表示發起者是否可以進行讀取、寫入或兩者皆可。
- **延遲**: `m_dmi_read_latency` 和 `m_dmi_write_latency` 告訴發起者每次存取需要計算多少時間。

## 3. 工作流程
1.  **請求**: 發起者在目標上呼叫 `get_direct_mem_ptr(addr, dmi_data)`。
2.  **授予**: 目標填寫 `dmi_data` 結構，如果該地址支援 DMI 則回傳 true。
3.  **存取**: 發起者使用原始指標讀取/寫入資料，並將指定的延遲加到其區域時間量子 (local time quantum) 中。
4.  **撤銷**: 目標呼叫 `invalidate_direct_mem_ptr(range)` 來撤銷存取權限 (例如，如果記憶體被重新映射)。

## 4. 關鍵重點
1.  **最佳化**: DMI 是指令集模擬器 (ISS) 從記憶體模型中以每秒數百萬條指令 (MIPS) 的速度獲取指令最有效的一種最佳化手段。
