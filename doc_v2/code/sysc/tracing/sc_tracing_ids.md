# sc_tracing_ids.h - 追蹤子系統錯誤訊息 ID

> 定義追蹤子系統使用的所有錯誤、警告、資訊訊息的唯一識別碼。ID 範圍為 700-799。

## 日常生活比喻

這個檔案就像醫院的「診斷代碼表」。每個代碼對應一種特定的「症狀」——當追蹤系統出問題時，它不會只說「壞了」，而是給出精確的代碼（例如 701 = 「無法開啟檔案」），方便工程師迅速定位問題。

## 概覽

透過 `SC_DEFINE_MESSAGE` 巨集定義訊息。每個訊息包含三個部分：
1. **符號常數**（如 `SC_ID_TRACING_FOPEN_FAILED_`）— 程式碼中使用
2. **數字 ID**（如 701）— 用於過濾和辨識
3. **預設訊息文字**（如 `"cannot open trace file for writing"`）— 人類可讀

## 訊息清單

| ID | 符號常數 | 訊息 | 嚴重程度 | 觸發時機 |
|----|---------|------|---------|---------|
| 701 | `SC_ID_TRACING_FOPEN_FAILED_` | cannot open trace file for writing | ERROR | 檔案開啟失敗（路徑錯誤、權限不足、磁碟空間不足） |
| 702 | `SC_ID_TRACING_TIMESCALE_DEFAULT_` | default timescale unit used for tracing | INFO | 使用者未設定時間刻度，自動使用 kernel 解析度 |
| 703 | `SC_ID_TRACING_TIMESCALE_UNIT_` | tracing timescale unit set | INFO | 使用者手動設定了追蹤時間單位 |
| 704 | `SC_ID_TRACING_VCD_DELTA_CYCLE_` | VCD delta cycle tracing with pseudo timesteps (1 unit) | INFO | 在 VCD 格式中啟用 delta cycle 追蹤 |
| 705 | `SC_ID_TRACING_INVALID_TIMESCALE_UNIT_` | invalid tracing timescale unit set | ERROR | 時間單位不是 10 的冪次 |
| 710 | `SC_ID_TRACING_OBJECT_IGNORED_` | object cannot not be traced | WARNING | 嘗試追蹤不支援的型別（觸發 `void*` 版本的 `sc_trace`） |
| 711 | `SC_ID_TRACING_OBJECT_NAME_FILTERED_` | traced object name filtered | INFO | 追蹤物件名稱被過濾（含非法字元） |
| 712 | `SC_ID_TRACING_INVALID_ENUM_VALUE_` | traced value of enumerated type undefined | WARNING | 列舉值超出定義範圍 |
| 713 | `SC_ID_TRACING_VCD_TIME_RESOLUTION_` | current kernel time is not representable in VCD time units | WARNING | 模擬時間無法用 VCD 的時間單位精確表示 |
| 714 | `SC_ID_TRACING_REVERSED_TIME_` | tracing cycle with duplicate or reversed time detected | WARNING | 偵測到時間倒退或重複的追蹤時間戳 |
| 715 | `SC_ID_TRACING_CLOSE_EMPTY_FILE_` | trace file closed before any cycles were traced, file not written | WARNING | 追蹤檔案關閉時從未記錄任何資料 |
| 720 | `SC_ID_TRACING_ALREADY_INITIALIZED_` | sc_trace_file already initialized | ERROR | 模擬已開始後嘗試修改時間刻度或新增追蹤物件 |

> 注意：ID 706-709 和 716-719 目前未使用，保留供未來擴充。

## SC_DEFINE_MESSAGE 巨集

```cpp
#define SC_DEFINE_MESSAGE(id, unused1, unused2) \
    namespace sc_core { extern SC_API const char id[]; }
```

這個巨集在標頭檔中只宣告外部常數字串。實際的字串定義在對應的 `.cpp` 檔案中（透過 `sysc/utils/sc_report.cpp` 的機制）。

`unused1`（數字 ID）和 `unused2`（訊息文字）在這個巨集展開中看似被忽略，但它們會被其他版本的 `SC_DEFINE_MESSAGE` 使用（在 `.cpp` 中定義時）。

## 使用情境

### 開發者常見問題對照

| 你遇到的問題 | 可能看到的訊息 ID |
|-------------|-----------------|
| 追蹤檔案沒有產生 | 701（開檔失敗）或 715（未記錄任何資料） |
| 時間刻度看起來不對 | 702（用了預設值）或 705（無效單位） |
| 某個訊號沒有出現在波形中 | 710（型別不支援）或 720（太晚加入） |
| VCD 檔案中時間戳有問題 | 713（解析度不足）或 714（時間倒退） |

## 相關檔案

- [sc_trace_file_base.md](sc_trace_file_base.md) — 使用這些 ID 報告錯誤的主要位置
- [sc_trace.md](sc_trace.md) — `sc_trace()` 中使用 `SC_ID_TRACING_OBJECT_IGNORED_`
- [sc_vcd_trace.md](sc_vcd_trace.md) — VCD 特有的訊息使用
- `sysc/utils/sc_report.h` — 錯誤回報機制
