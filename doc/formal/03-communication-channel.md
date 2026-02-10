# 2026-02-08-communication-channel

## 1. 頻道 (Channel) 的雙重分類
在 SystemC 中，**頻道 (Channel)** 是實現 `sc_interface` 的具體類別，負責提供通訊服務。頻道分為兩大類：

| 類型 | 基類 | 特性 | 範例 |
|------|------|------|------|
| **基本頻道 (Primitive)** | `sc_prim_channel` | 無內部結構，直接操作數據 | `sc_signal`, `sc_fifo` |
| **階層式頻道 (Hierarchical)** | `sc_module` | 包含子模組與內部連接 | 自定義總線、仲裁器 |

## 2. 基本頻道 (Primitive Channel) 深度解析
位於 `src/sysc/communication/sc_prim_channel.h`。

### 核心設計
```cpp
class sc_prim_channel : public sc_object {
    virtual void update();  // 在 Update Phase 執行
    void request_update();  // 請求 update() 被調用
};
```

### Update 機制 - 解決 Race Condition
基本頻道使用 **請求-更新 (Request-Update)** 分離模式：

1. **Write Phase** (Evaluation Phase): 
   - 使用者呼叫 `signal.write(new_value)`。
   - 內部僅記錄新值到暫存區 (New Value)。
   - 呼叫 `request_update()` 將自己加入 **Update List**。

2. **Update Phase** (Delta Cycle 結束前):
   - 核心依序執行 Update List 中所有頻道的 `update()`。
   - `update()` 將暫存值寫入當前值，並觸發 `value_changed_event`。

### 為何需要分離？
避免在同一 Delta Cycle 內產生不可預測的執行順序依賴 (Race Condition)：
```cpp
// 若無 Update 機制，這兩行的執行順序會影響結果
sig1.write(10);  // 若立即生效
sig2.write(sig1.read());  // 可能讀到 10 或舊值，取決於順序
```
有了 Update 機制，所有 `write()` 都在同一個 Delta 內完成，所有 `read()` 都讀到舊值，確保確定性。

## 3. 階層式頻道 (Hierarchical Channel)
階層式頻道本質上就是 **`sc_module`**，但它實作了一個或多個 `sc_interface`。

### 實作方式
```cpp
class my_bus : public sc_module, public sc_interface {
    SC_HAS_PROCESS(my_bus);
    
    sc_port<sc_signal_in_if<int>> addr_port;  // 子模組/端口
    sc_port<sc_signal_in_if<int>> data_port;
    
    void arbitration_thread();  // 內部處理邏輯
};
```

### 與基本頻道的關鍵差異
| 特性 | Primitive Channel | Hierarchical Channel |
|------|-------------------|---------------------|
| 內部結構 | 無 | 可包含子模組、Port、Process |
| 更新時機 | 固定的 Update Phase | 依賴內部 Process 的動態行為 |
| 用途 | 簡單資料傳遞 | 複雜協議實作 (AMBA, AXI) |
| 繼承 | `sc_prim_channel` | `sc_module` + `sc_interface` |

## 4. sc_prim_channel_registry - 更新管理
`sc_prim_channel_registry` 管理所有基本頻道的更新請求：
- **`m_update_list_p`**: 指向待更新頻道的鏈表。
- **`perform_update()`**: 在 Update Phase 遍歷鏈表並呼叫 `update()`。
- **異步更新支援**: `async_request_update()` 允許模擬外部硬體事件。

## 5. 原始碼參考
- `src/sysc/communication/sc_prim_channel.h`: 基本頻道基類與 Update 機制。
- `src/sysc/communication/sc_signal.h`: `sc_signal` 作為 Primitive Channel 的實作範例。
- `src/sysc/kernel/sc_module.h`: 階層式頻道的基礎。

---
*Source: ref/systemc/src/sysc/communication/*
