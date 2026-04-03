# sc_communication_ids -- 通訊子系統的錯誤/警告訊息 ID

## 概述

`sc_communication_ids.h` 定義了通訊子系統中所有錯誤和警告訊息的唯一識別碼。這些 ID 用於 `SC_REPORT_ERROR`、`SC_REPORT_WARNING` 等巨集，讓使用者可以辨識、過濾和處理特定類型的錯誤。

**原始檔案：** `sc_communication_ids.h`（僅標頭檔）

## 日常比喻

就像醫院的疾病分類代碼（ICD codes）一樣，每種錯誤都有一個唯一的編號。當你收到錯誤訊息時，可以透過編號快速查閱原因和解決方法。

## 訊息 ID 列表

所有通訊子系統的訊息 ID 範圍為 **100-199**。

### 埠相關 (100, 107-112)

| ID | 常數名稱 | 編號 | 說明 |
|----|---------|------|------|
| 100 | `SC_ID_PORT_OUTSIDE_MODULE_` | 100 | 埠在模組外部被建立 |
| 107 | `SC_ID_BIND_IF_TO_PORT_` | 107 | 綁定介面到埠失敗 |
| 108 | `SC_ID_BIND_PORT_TO_PORT_` | 108 | 綁定父埠到埠失敗 |
| 109 | `SC_ID_COMPLETE_BINDING_` | 109 | 完成綁定失敗 |
| 110 | `SC_ID_INSERT_PORT_` | 110 | 插入埠失敗 |
| 111 | `SC_ID_REMOVE_PORT_` | 111 | 移除埠失敗 |
| 112 | `SC_ID_GET_IF_` | 112 | 取得介面失敗 |

### 時脈相關 (101-103, 125, 128)

| ID | 常數名稱 | 編號 | 說明 |
|----|---------|------|------|
| 101 | `SC_ID_CLOCK_PERIOD_ZERO_` | 101 | sc_clock 週期為零 |
| 102 | `SC_ID_CLOCK_HIGH_TIME_ZERO_` | 102 | sc_clock 高電位時間為零 |
| 103 | `SC_ID_CLOCK_LOW_TIME_ZERO_` | 103 | sc_clock 低電位時間為零 |
| 125 | `SC_ID_ATTEMPT_TO_WRITE_TO_CLOCK_` | 125 | 嘗試寫入 sc_clock 的值 |
| 128 | `SC_ID_ATTEMPT_TO_BIND_CLOCK_TO_OUTPUT_` | 128 | 嘗試將 sc_clock 綁定到輸出埠 |

### FIFO 相關 (104-106)

| ID | 常數名稱 | 編號 | 說明 |
|----|---------|------|------|
| 104 | `SC_ID_MORE_THAN_ONE_FIFO_READER_` | 104 | sc_fifo 不能有多個讀取者 |
| 105 | `SC_ID_MORE_THAN_ONE_FIFO_WRITER_` | 105 | sc_fifo 不能有多個寫入者 |
| 106 | `SC_ID_INVALID_FIFO_SIZE_` | 106 | sc_fifo 大小至少為 1 |

### 通道相關 (113-116)

| ID | 常數名稱 | 編號 | 說明 |
|----|---------|------|------|
| 113 | `SC_ID_INSERT_PRIM_CHANNEL_` | 113 | 插入原始通道失敗 |
| 114 | `SC_ID_REMOVE_PRIM_CHANNEL_` | 114 | 移除原始通道失敗 |
| 115 | `SC_ID_MORE_THAN_ONE_SIGNAL_DRIVER_` | 115 | sc_signal 不能有多個驅動器 |
| 116 | `SC_ID_NO_DEFAULT_EVENT_` | 116 | 通道沒有預設事件 |

### 事件尋找器與解析相關 (117-118)

| ID | 常數名稱 | 編號 | 說明 |
|----|---------|------|------|
| 117 | `SC_ID_RESOLVED_PORT_NOT_BOUND_` | 117 | resolved port 未綁定到 resolved signal |
| 118 | `SC_ID_FIND_EVENT_` | 118 | 尋找事件失敗 |

### 信號量 (119)

| ID | 常數名稱 | 編號 | 說明 |
|----|---------|------|------|
| 119 | `SC_ID_INVALID_SEMAPHORE_VALUE_` | 119 | sc_semaphore 初始值必須 >= 0 |

### Export 相關 (120-124, 126)

| ID | 常數名稱 | 編號 | 說明 |
|----|---------|------|------|
| 120 | `SC_ID_SC_EXPORT_HAS_NO_INTERFACE_` | 120 | sc_export 沒有介面 |
| 121 | `SC_ID_INSERT_EXPORT_` | 121 | 插入 sc_export 失敗 |
| 122 | `SC_ID_EXPORT_OUTSIDE_MODULE_` | 122 | sc_export 在模組外部被建立 |
| 123 | `SC_ID_SC_EXPORT_NOT_REGISTERED_` | 123 | 移除未註冊的 sc_export |
| 124 | `SC_ID_SC_EXPORT_NOT_BOUND_AFTER_CONSTRUCTION_` | 124 | sc_export 建構結束後未綁定 |
| 126 | `SC_ID_SC_EXPORT_ALREADY_BOUND_` | 126 | sc_export 已經綁定 |

### 其他 (127, 129)

| ID | 常數名稱 | 編號 | 說明 |
|----|---------|------|------|
| 127 | `SC_ID_INVALID_HIERARCHICAL_BIND_` | 127 | 多目標 socket 綁定錯誤 |
| 129 | `SC_ID_INSERT_STUB_` | 129 | 插入 sc_stub 失敗 |

## 定義機制

```cpp
#define SC_DEFINE_MESSAGE(id, unused1, unused2) \
    namespace sc_core { extern SC_API const char id[]; }
```

每個訊息 ID 被定義為一個 `extern const char[]` 陣列。實際的字串內容在對應的 `.cpp` 檔案中定義。巨集的第二個參數是編號（用於唯一性），第三個參數是人類可讀的描述。

## 使用方式

在程式碼中透過 `SC_REPORT_ERROR` 或 `SC_REPORT_WARNING` 使用：

```cpp
// example from sc_interface.cpp
SC_REPORT_WARNING( SC_ID_NO_DEFAULT_EVENT_, 0 );

// example from sc_port.cpp
SC_REPORT_ERROR( SC_ID_BIND_IF_TO_PORT_, "simulation running" );
```

使用者可以透過 `sc_report_handler` 來自訂錯誤處理行為，例如將某些錯誤轉為警告，或完全抑制某些訊息。

## 設計重點

### 為什麼使用字串常數而非枚舉？

字串常數可以包含更多語意資訊，而且在錯誤報告中直接可讀。`SC_REPORT_ERROR` 系統會同時輸出 ID 字串和附加訊息，方便除錯。

### 編號範圍

不同子系統使用不同的編號範圍：
- 100-199: 通訊子系統（本檔案）
- 其他範圍由 kernel、datatypes 等子系統使用

## 相關檔案

- `sc_report.h` - 錯誤報告基礎設施
- `sc_port.cpp` - 使用這些 ID 報告綁定錯誤
- `sc_export.cpp` - 使用這些 ID 報告匯出錯誤
- `sc_signal.cpp` - 使用這些 ID 報告訊號錯誤
- `sc_clock.cpp` - 使用這些 ID 報告時脈錯誤
