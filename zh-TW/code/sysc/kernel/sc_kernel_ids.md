# sc_kernel_ids.h - 核心錯誤與警告訊息 ID

## 概觀

`sc_kernel_ids.h` 集中定義了 SystemC 核心（kernel）所有的錯誤和警告訊息 ID。每個 ID 對應一個數字代碼和一段人類可讀的描述文字。這些 ID 被 `SC_REPORT_ERROR`、`SC_REPORT_WARNING` 等巨集使用來產生診斷訊息。

## 為什麼需要這個檔案？

就像醫院有統一的疾病代碼系統（ICD），SystemC 也需要一套統一的錯誤代碼。這樣做的好處：

1. **可程式化處理**：可以用數字 ID 來過濾或特殊處理某些錯誤
2. **統一管理**：所有錯誤訊息集中在一個地方，方便翻譯和修改
3. **避免重複**：每個錯誤有唯一的 ID，不會混淆

## 訊息定義機制

```cpp
#ifndef SC_DEFINE_MESSAGE
#define SC_DEFINE_MESSAGE(id, unused1, unused2) \
    namespace sc_core { extern SC_API const char id[]; }
#endif
```

`SC_DEFINE_MESSAGE` 巨集有三個參數：
- `id`：C++ 變數名稱（如 `SC_ID_COROUTINE_ERROR_`）
- `unused1`：數字代碼（如 518）
- `unused2`：錯誤描述字串

在 header 中，這個巨集只宣告一個 `extern const char[]`。實際的字串定義在別處（通常由報告系統自動處理）。

## 訊息分類總覽

訊息 ID 範圍為 500-578，按主題分組如下：

### 型別與運算錯誤（500-504）

| ID | 代碼 | 說明 |
|----|------|------|
| `SC_ID_NO_BOOL_RETURNED_` | 500 | 運算子未回傳 boolean |
| `SC_ID_NO_INT_RETURNED_` | 501 | 運算子未回傳 int |
| `SC_ID_NO_SC_LOGIC_RETURNED_` | 502 | 運算子未回傳 sc_logic |
| `SC_ID_OPERAND_NOT_SC_LOGIC_` | 503 | 運算元不是 sc_logic |
| `SC_ID_OPERAND_NOT_BOOL_` | 504 | 運算元不是 bool |

### 物件與命名（505-506, 532-534）

| ID | 代碼 | 說明 |
|----|------|------|
| `SC_ID_INSTANCE_EXISTS_` | 505 | 物件已存在 |
| `SC_ID_ILLEGAL_CHARACTERS_` | 506 | 名稱中有非法字元 |
| `SC_ID_GEN_UNIQUE_NAME_` | 532 | 無法從空字串生成唯一名稱 |
| `SC_ID_MODULE_NAME_STACK_EMPTY_` | 533 | 模組名稱堆疊為空 |
| `SC_ID_NAME_EXISTS_` | 534 | 名稱已存在 |

### 模組建構（509-513, 569）

| ID | 代碼 | 說明 |
|----|------|------|
| `SC_ID_END_MODULE_NOT_CALLED_` | 509 | 模組建構未正確完成（忘記 `sc_module_name` 參數） |
| `SC_ID_HIER_NAME_INCORRECT_` | 510 | 階層名稱可能不正確 |
| `SC_ID_SET_STACK_SIZE_` | 511 | `set_stack_size()` 只能用於 `SC_THREAD`/`SC_CTHREAD` |
| `SC_ID_SC_MODULE_NAME_USE_` | 512 | `sc_module_name` 使用不正確 |
| `SC_ID_SC_MODULE_NAME_REQUIRED_` | 513 | 建構子需要 `sc_module_name` 參數 |
| `SC_ID_BAD_SC_MODULE_CONSTRUCTOR_` | 569 | 已棄用的建構子方式 |

### 時間設定（514-516, 567）

| ID | 代碼 | 說明 |
|----|------|------|
| `SC_ID_SET_TIME_RESOLUTION_` | 514 | 設定時間解析度失敗 |
| `SC_ID_SET_DEFAULT_TIME_UNIT_` | 515 | 設定預設時間單位失敗 |
| `SC_ID_DEFAULT_TIME_UNIT_CHANGED_` | 516 | 預設時間單位被改為時間解析度 |
| `SC_ID_TIME_CONVERSION_FAILED_` | 567 | sc_time 轉換失敗 |

### Process 控制（519-528, 537-543, 556-561, 563-564, 572-574）

| ID | 代碼 | 說明 |
|----|------|------|
| `SC_ID_WAIT_NOT_ALLOWED_` | 519 | `wait()` 只能在 `SC_THREAD`/`SC_CTHREAD` 中使用 |
| `SC_ID_NEXT_TRIGGER_NOT_ALLOWED_` | 520 | `next_trigger()` 只能在 `SC_METHOD` 中使用 |
| `SC_ID_HALT_NOT_ALLOWED_` | 522 | `halt()` 只能在 `SC_CTHREAD` 中使用 |
| `SC_ID_DONT_INITIALIZE_` | 524 | `dont_initialize()` 對 `SC_CTHREAD` 無效 |
| `SC_ID_WAIT_N_INVALID_` | 525 | `wait(n)` 的 n 必須大於 0 |
| `SC_ID_WAIT_DURING_UNWINDING_` | 537 | unwinding 期間不能呼叫 `wait()` |
| `SC_ID_RETHROW_UNWINDING_` | 539 | kill/reset 時未重新拋出 `sc_unwind_exception` |
| `SC_ID_PROCESS_ALREADY_UNWINDING_` | 540 | unwinding 中忽略 kill/reset |

### 模擬控制（544-549, 554）

| ID | 代碼 | 說明 |
|----|------|------|
| `SC_ID_SIMULATION_TIME_OVERFLOW_` | 544 | 模擬時間溢位 |
| `SC_ID_SIMULATION_STOP_CALLED_TWICE_` | 545 | `sc_stop` 被呼叫兩次 |
| `SC_ID_SIMULATION_START_AFTER_STOP_` | 546 | 停止後又呼叫 `sc_start` |
| `SC_ID_SIMULATION_START_AFTER_ERROR_` | 548 | 錯誤後嘗試重啟模擬 |
| `SC_ID_SIMULATION_UNCAUGHT_EXCEPTION_` | 549 | 未捕獲的例外 |

### 其他

| ID | 代碼 | 說明 |
|----|------|------|
| `SC_ID_INCONSISTENT_API_CONFIG_` | 517 | 偵測到不一致的函式庫配置 |
| `SC_ID_COROUTINE_ERROR_` | 518 | 協程套件錯誤 |
| `SC_ID_IMMEDIATE_NOTIFICATION_` | 521 | update phase 或 elaboration 中不允許立即通知 |
| `SC_ID_STAGE_CALLBACK_REGISTER_` | 552 | 階段回呼註冊相關 |
| `SC_ID_STAGE_CALLBACK_FORBIDDEN_` | 553 | 階段回呼中的禁止操作 |
| `SC_ID_INSERT_INITIALIZER_FN_` | 568 | 插入初始化函式失敗 |
| `SC_ID_UNMATCHED_SUSPENDABLE_` | 576 | 不匹配的 suspendable/unsuspendable 請求 |

## 使用方式

```cpp
// In SystemC source code:
SC_REPORT_ERROR(SC_ID_WAIT_NOT_ALLOWED_, "");
SC_REPORT_WARNING(SC_ID_SIMULATION_STOP_CALLED_TWICE_, "");
```

## 相關檔案

- `sysc/utils/sc_report.h` - 報告系統（定義 `SC_DEFINE_MESSAGE` 巨集）
- `sc_simcontext.h` - 使用這些 ID 產生模擬錯誤
- `sc_process_b.h` - 使用 process 相關的錯誤 ID
