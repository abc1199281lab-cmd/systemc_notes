# 2026-02-08-communication-signal

## 1. sc_signal 概論
`sc_signal<T>` 是 SystemC 中最常用的基本頻道 (Primitive Channel)，用於模擬硬體訊號線 (Wire) 的行為。它繼承自 `sc_prim_channel`，並實作了 `sc_signal_in_if<T>` 與 `sc_signal_write_if<T>` 介面。

## 2. Write / Update 分離機制深度解析
`sc_signal` 的核心設計是將資料寫入與實際更新分離到不同的模擬階段：

### 雙值儲存 (Double Buffering)
```cpp
template<typename T, typename POL>
class sc_signal : public sc_signal_channel {
    T m_cur;   // 當前值 (Current Value) - 供 read() 使用
    T m_new;   // 新值 (New Value) - 供 write() 寫入
};
```

### 兩階段操作
| 階段 | 方法 | 行為 | 模擬時間點 |
|------|------|------|---------|
| **Write** | `write(T val)` | 寫入 `m_new`，呼叫 `request_update()` | Evaluation Phase |
| **Update** | `update()` (virtual) | `m_cur = m_new`，觸發事件 | Update Phase |
| **Read** | `read()` / `operator T()` | 返回 `m_cur` | 任意時間 |

### 為何需要分離？避免 Race Condition
在同一個 Delta Cycle 中，不論 `write()` 呼叫順序如何，所有讀取操作都看到**一致的舊值**，直到 Update Phase 才統一更新：

```cpp
// 範例：組合邏輯迴路
SC_METHOD(comb_logic);
sensitive << a << b;

void comb_logic() {
    sig.write(a.read() & b.read());  // 寫入新值到 m_new
    // 此時 sig.read() 仍返回舊的 m_cur！
}
```
若無分離機制，`write()` 立即生效會導致執行順序依賴性 (Ordering Dependency)。

## 3. Writer Policy - 寫入策略檢查
位於 `src/sysc/communication/sc_writer_policy.h`。

SystemC 透過模板策略參數 `POL` 控制寫入行為：

| 策略 | 定義 | 行為 |
|------|------|------|
| `SC_ONE_WRITER` | 預設 | 僅允許單一行程寫入，多寫入報錯 |
| `SC_MANY_WRITERS` | 顯式放寬 | 允許多個行程寫入，但同一 Delta 內的值必須相同 |
| `SC_UNCHECKED_WRITERS` | 無檢查 | 完全開放，不檢查衝突 (非標準) |

### 實作機制
```cpp
struct sc_writer_policy_check_write {
    sc_process_handle m_writer_p;  // 記錄第一個寫入者
    
    bool check_write(sc_object* target, bool value_changed) {
        if (m_writer_p != current_writer) {
            // 多寫入者檢測，報告錯誤
            sc_signal_invalid_writer(target, m_writer_p, current_writer, ...);
        }
    }
};
```

## 4. 事件觸發 (Event Notification)
`sc_signal` 提供三種事件供敏感度設定：
- **`value_changed_event()`**: 值改變時觸發（最常用）。
- **`posedge_event()`**: 當 `T=bool/sc_logic` 且 0→1 跳變時觸發。
- **`negedge_event()`**: 當 `T=bool/sc_logic` 且 1→0 跳變時觸發。

觸發發生在 **Update Phase** 的 `update()` 方法中，僅當 `m_cur != m_new` 時。

## 5. 原始碼參考
- `src/sysc/communication/sc_signal.h`: `sc_signal<T>` 主類別。
- `src/sysc/communication/sc_writer_policy.h`: 寫入策略檢查邏輯。
- `src/sysc/communication/sc_signal_ifs.h`: `sc_signal_in_if` 等介面定義。

---
*Source: ref/systemc/src/sysc/communication/*
