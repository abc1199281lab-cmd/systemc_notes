# at_extension_optional -- 原始碼詳解

> **原始碼路徑**: `ref/systemc/examples/tlm/at_extension_optional/`

## 軟體類比總覽

Optional extension 就像 **HTTP 的 optional headers**：

```
標準 HTTP 請求 (generic payload):
  GET /api/data HTTP/1.1
  Host: server.example.com
  Content-Type: application/json

帶有 optional header (extension) 的請求:
  GET /api/data HTTP/1.1
  Host: server.example.com
  Content-Type: application/json
  X-Initiator-Id: client-101        <-- optional extension!
  X-Request-Priority: high          <-- another optional extension!
```

- Server A（支援 `X-Initiator-Id`）：讀取並記錄
- Server B（不支援）：正常處理請求，完全忽略不認識的 header

## 系統架構

這個範例的獨特之處在於它**混合了兩種不同的 target**：

```
example_system_top
  |-- SimpleBusAT<2, 2>       m_bus
  |-- at_target_4_phase       m_at_target_4_phase_1   (ID=201) -- 支援 extension
  |-- at_target_2_phase       m_at_target_2_phase_2   (ID=202) -- 不支援 extension
  |-- initiator_top           m_initiator_1            (ID=101)
  |-- initiator_top           m_initiator_2            (ID=102)
```

這就像一個系統中同時有「新版 API server」和「舊版 API server」，client 送出的 optional header 只有新版 server 會理解。

## Extension 定義：extension_initiator_id

Extension 定義在共用模組 `common/include/extension_initiator_id.h`：

```cpp
class extension_initiator_id
: public tlm::tlm_extension<extension_initiator_id>
{
  public:
    std::string m_initiator_id;   // 儲存 initiator 的識別字串

    void copy_from(const tlm_extension_base &extension);
    tlm::tlm_extension_base* clone() const;
};
```

### 軟體對應

| Extension 方法 | 軟體類比 | 用途 |
| --- | --- | --- |
| `m_initiator_id` | Header value string | 儲存 initiator 識別資訊 |
| `copy_from()` | Deep copy constructor | 當 payload 被複製時（例如 bus routing），extension 也要複製 |
| `clone()` | `Object.clone()` | 建立 extension 的獨立副本 |

### 為什麼需要 clone 和 copy_from？

在 TLM 系統中，bus 可能需要複製 payload（例如做 broadcast）。如果 extension 只是淺拷貝，多個副本會共用同一份資料，導致 race condition。這就像在多執行緒環境中，你需要確保每個 thread 有自己的 request object 副本。

## Target 如何使用 Extension

在 `at_target_4_phase.cpp` 中（需要定義 `USING_EXTENSION_OPTIONAL` 編譯旗標）：

```cpp
#if (defined(USING_EXTENSION_OPTIONAL))
  extension_initiator_id *extension_pointer;
  gp.get_extension(extension_pointer);   // 嘗試取得 extension

  if (extension_pointer) {
    // extension 存在，可以使用
    msg << "data: " << extension_pointer->m_initiator_id;
  }
  // 如果 extension_pointer 為 null，就完全忽略
#endif
```

軟體對應：

```python
# Python 中讀取 optional header
initiator_id = request.headers.get("X-Initiator-Id")  # 可能為 None
if initiator_id:
    logger.info(f"Request from: {initiator_id}")
# 即使沒有這個 header，正常邏輯繼續執行
```

## Target 比較

| 面向 | at_target_4_phase (ID=201) | at_target_2_phase (ID=202) |
| --- | --- | --- |
| 協定 | 4-phase | 2-phase |
| Extension 支援 | 有（條件編譯） | 無 |
| 收到帶 extension 的 payload | 讀取並記錄 | 完全忽略 |
| 功能影響 | 無（extension 僅用於 logging） | 無 |

## Initiator 端

`initiator_top` 的結構與其他範例完全相同。Extension 的附加（`set_extension`）是在 `select_initiator` 或 `traffic_generator` 中完成的（取決於編譯設定）。

## 設計模式：Optional Extension 的正確使用方式

### 適合用 extension 的場景

| 場景 | 例子 |
| --- | --- |
| Debug / logging 資訊 | Initiator ID、transaction 序號 |
| 效能提示 | Priority、cache hint |
| 安全標記 | Access permission、security domain |

### 不適合用 extension 的場景

| 場景 | 應該用什麼 |
| --- | --- |
| 必要的地址資訊 | `generic_payload.set_address()` |
| 讀/寫指令 | `generic_payload.set_command()` |
| 資料 payload | `generic_payload.set_data_ptr()` |

原則：如果缺少這個資訊 target 就無法正常運作，那它不該是 extension，而應該是 generic payload 的標準欄位。

## 重點整理

| 概念 | 說明 |
| --- | --- |
| **Optional extension** | 附加在 generic payload 上的可選 metadata |
| **互操作性** | 不支援 extension 的 target 可以安全地忽略它 |
| **條件編譯** | 用 `USING_EXTENSION_OPTIONAL` 控制是否啟用 extension 處理 |
| **clone/copy_from** | Extension 必須支援深拷貝，因為 bus 可能需要複製 payload |
| **混合 target** | 同一系統中可以混用支援和不支援 extension 的 target |
