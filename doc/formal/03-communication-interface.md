# 2026-02-08-communication-interface

## 1. SystemC 通訊抽象層次概論
SystemC 採用分層的通訊模型，將「介面定義」與「實作細節」分離。核心抽象由三個關鍵類別構成：
- **`sc_interface`**: 定義服務介面（抽象基類）。
- **`sc_port`**: 模組對外的連接點（Client）。
- **`sc_channel`**: 實際提供服務的通道（Server）。

這種分離允許模組僅依賴於抽象介面，而無需知道具體使用哪種通道實作。

## 2. sc_interface - 抽象介面基類
位於 `src/sysc/communication/sc_interface.h`。

### 核心設計
```cpp
class sc_interface {
    virtual void register_port(sc_port_base& port, const char* if_typename);
    virtual const sc_event& default_event() const;
};
```

- **純虛擬基類**: 所有具體通道（如 `sc_signal`, `sc_fifo`）都繼承此類。
- **Port 註冊**: `register_port()` 允許通道追蹤哪些 Port 連接到它。
- **預設事件**: `default_event()` 提供統一的事件觸發介面，用於敏感度列表。

### 繼承警告
源碼註釋明確警告：**必須使用虛擬繼承** (`virtual public sc_interface`)，以防止在多重繼承場景中出現二義性。

## 3. sc_port - 埠（Port）基類
位於 `src/sysc/communication/sc_port.h`。

### 核心職責
Port 是模組（`sc_module`）與外部世界通訊的「窗口」：
- **型別安全**: 模板參數 `IF` 確保編譯期檢查介面型別。
- **綁定策略 (Policy)**: 
  - `SC_ONE_OR_MORE_BOUND`: 至少一個綁定（預設）。
  - `SC_ZERO_OR_MORE_BOUND`: 可選綁定。
  - `SC_ALL_BOUND`: 所有埠必須綁定。

### 綁定時機限制
源碼中有明確警告：
```cpp
// !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
//  BEWARE: Ports can only be created and bound during elaboration.
// !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
```
Port 只能在 **Elaboration 階段**（構造函式、`before_end_of_elaboration`）創建和綁定。

## 4. 綁定機制 (Binding) 解析
綁定是建立 Port 與 Channel 之間連接的過程：

### A. 顯式綁定 (Explicit Binding)
在建構子中使用 `port.bind(channel)`：
```cpp
module.port(signal);  // 使用 operator() 語法糖
```

### B. 位置綁定 (Positional Binding)
使用 `operator<<` 按位置順序綁定：
```cpp
module << signal1 << signal2;  // 位置對應
```

### C. 階層綁定 (Hierarchical Binding)
上層模組的 Port 可以綁定到下層模組的 Port，形成通訊路徑的「接力」。

## 5. 動態敏感度與 Port
Port 通過 `make_sensitive()` 方法支援敏感度設定：
- 當 Channel 的 `default_event()` 觸發時，所有敏感行程被喚醒。
- 這是 `sensitive << port;` 語法的底層實現。

## 6. 原始碼參考
- `src/sysc/communication/sc_interface.h`: 抽象介面定義。
- `src/sysc/communication/sc_port.h`: Port 基類與綁定邏輯。
- `src/sysc/communication/sc_port_registry.h`: 管理所有 Port 實例。

---
*Source: ref/systemc/src/sysc/communication/*
