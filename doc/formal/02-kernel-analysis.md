# SystemC Kernel 模組深入分析

> **建立日期**: 2026-02-08  
> **版本**: SystemC 3.0.2  
> **對應 Phase**: Phase 2

## 概述

本文檔深入分析 SystemC 核心模組 (kernel module)，包含模擬器上下文、行程機制、事件系統、模組管理和協程實現。

---

## 1. sc_simcontext - 模擬器上下文

### 1.1 什麼是 sc_simcontext？

`sc_simcontext` 是 **SystemC 模擬器的核心大腦**，負責管理整個模擬的生命週期。可以把它想像成一個「模擬器控制器」，協調所有元件的運作。

### 1.2 類別結構

```cpp
class sc_simcontext {
    // 核心管理器 (Registry)
    sc_object_manager*          m_object_manager;
    sc_module_registry*         m_module_registry;
    sc_port_registry*           m_port_registry;
    sc_export_registry*         m_export_registry;
    sc_prim_channel_registry*   m_prim_channel_registry;
    sc_stage_callback_registry* m_stage_cb_registry;
    
    // 行程管理
    sc_process_table*           m_process_table;
    sc_curr_proc_info           m_curr_proc_info;
    sc_runnable*                m_runnable;
    
    // 事件管理
    std::vector<sc_event*>      m_delta_events;
    sc_ppq<sc_event_timed*>*    m_timed_events;
    
    // 時間管理
    sc_time_params*             m_time_params;
    sc_time                     m_curr_time;
    sc_dt::uint64               m_delta_count;
    
    // 執行狀態
    sc_status                   m_simulation_status;
    sc_stage                    m_stage;
    execution_phases            m_execution_phase;
    bool                        m_ready_to_simulate;
    bool                        m_elaboration_done;
};
```

### 1.3 執行階段 (Execution Phases)

```
SystemC 模擬週期

phase_initialize: 初始化階段
  - elaboration: 建立模組層級結構
  - binding: 連接埠與通道
  - before_end_of_elaboration 回調

phase_evaluate: 評估階段 (執行行程)
  - 執行所有可運行 (runnable) 的行程
  - 更新訊號請求

phase_update: 更新階段
  - 執行訊號的 update() 方法

phase_notify: 通知階段
  - 觸發事件，標記下一輪可執行行程
```

### 1.4 關鍵方法

| 方法 | 功能 |
|------|------|
| `initialize()` | 初始化模擬環境 |
| `simulate()` | 執行模擬 |
| `stop()` | 停止模擬 |
| `elaborate()` | 執行細化階段 |
| `prepare_to_simulate()` | 模擬前準備 |
| `crunch()` | 執行一個 delta 週期 |
| `time_stamp()` | 取得當前模擬時間 |
| `delta_count()` | 取得 delta 計數 |

### 1.5 RTL 知識補充：為什麼需要 Delta Cycle？

在硬體模擬中，Delta Cycle 解決「零時間延遲下的順序問題」：

```verilog
always @(posedge clk) begin
    a <= b;  // 非阻塞賦值
    c <= a;  // 這裡的 a 還是舊值！
end
```

Delta Cycle 讓 SystemC 能在同一個模擬時間點處理「先寫後讀」的順序，確保訊號更新的正確性。

---

## 2. sc_process_b - 行程基礎類別

### 2.1 行程類型

SystemC 有三種行程類型：

```cpp
enum sc_curr_proc_kind {
    SC_NO_PROC_,
    SC_METHOD_PROC_,
    SC_THREAD_PROC_,
    SC_CTHREAD_PROC_
};
```

### 2.2 行程狀態 (State Machine)

```
ps_normal -> ps_bit_ready_to_run (準備執行)
                -> ps_bit_disabled (禁用)
                -> ps_bit_suspended (暫停)
                -> ps_bit_zombie (已終止)
```

### 2.3 sc_process_b 核心成員

```cpp
class sc_process_b : public sc_object_host {
    sc_curr_proc_kind   m_process_kind;
    int                 m_state;
    int                 proc_id;
    sc_entry_func       m_semantics_method_p;
    sc_process_host*    m_semantics_host_p;
    bool                m_is_thread;
    bool                m_has_stack;
    trigger_t           m_trigger_type;
    std::vector<const sc_event*> m_static_events;
    const sc_event*     m_event_p;
    const sc_event_list* m_event_list_p;
    int                 m_event_count;
    bool                m_timed_out;
    sc_event*           m_timeout_event_p;
    int                 m_active_areset_n;
    int                 m_active_reset_n;
    bool                m_sticky_reset;
    sc_event*           m_reset_event_p;
    std::vector<sc_reset*> m_resets;
    bool                m_unwinding;
    process_throw_type  m_throw_status;
    sc_throw_it_helper* m_throw_helper_p;
    int                 m_references_n;
    sc_process_b*       m_exist_p;
    sc_process_b*       m_runnable_p;
};
```

### 2.4 觸發類型 (trigger_t)

```cpp
enum trigger_t {
    STATIC,           // 靜態敏感度
    EVENT,            // 單一事件
    OR_LIST,          // 事件或列表
    AND_LIST,         // 事件與列表
    TIMEOUT,
    EVENT_TIMEOUT,
    OR_LIST_TIMEOUT,
    AND_LIST_TIMEOUT
};
```

---

## 3. sc_method_process - 方法行程

### 3.1 什麼是 Method Process？

Method Process 是 **不能暫停** 的行程，執行完畢後立即返回。類似 Verilog 中的 always_comb 組合邏輯。

### 3.2 特性

- 不能呼叫 wait() - 會編譯錯誤
- 使用 next_trigger() 設定下一次觸發
- 立即執行，不佔用堆疊
- 適合組合邏輯建模

### 3.3 執行流程

```
觸發事件發生
     |
     v
trigger_static() -> 檢查狀態、觸發類型
     |
     v 加入可運行佇列
run_process() -> 執行語義方法
     |
     v
等待下一次觸發
```

---

## 4. sc_thread_process - 執行緒行程

### 4.1 什麼是 Thread Process？

Thread Process 是 **可以暫停** 的行程，可以呼叫 wait() 等待事件。類似 Verilog 中的 always 時序邏輯。

### 4.2 特性

- 可以呼叫 wait() - 暫停執行
- 有獨立的堆疊 (stack)
- 使用協程 (coroutine) 實現暫停/恢復
- 適合時序邏輯建模

### 4.3 關鍵機制：suspend_me()

```cpp
inline void sc_thread_process::suspend_me() {
    simc_p->cor_pkg()->yield(cor_p);
    
    switch(m_throw_status) {
        case THROW_ASYNC_RESET:
        case THROW_SYNC_RESET:
            throw sc_unwind_exception(this, true);
        case THROW_USER:
            m_throw_helper_p->throw_it();
        case THROW_KILL:
            throw sc_unwind_exception(this, false);
    }
}
```

### 4.4 wait() 多載方法

```cpp
void wait(const sc_event&);
void wait(const sc_event_or_list&);
void wait(const sc_event_and_list&);
void wait(const sc_time&);
void wait(const sc_time&, const sc_event&);
void wait_cycles(int n);
```

---

## 5. sc_cthread_process - 時脈執行緒行程

### 5.1 特性

- 繼承自 sc_thread_process
- 與時脈邊緣綁定
- 支援 halt() 永久暫停

### 5.2 使用方式

```cpp
SC_CTHREAD(thread_func, clk.pos());
```

---

## 6. sc_event - 事件機制

### 6.1 事件類型

```cpp
enum notify_t { NONE, DELTA, TIMED };
```

### 6.2 通知方式

```cpp
void notify();                  // 立即通知 (delta)
void notify(const sc_time&);    // 定時通知
void notify(double, sc_time_unit);
void notify_delayed();          // 下一個 delta 延遲通知
```

---

## 7. sc_cor - 協程機制

### 7.1 協程抽象類別

```cpp
class sc_cor {
    virtual void stack_protect(bool enable) {}
};

class sc_cor_pkg {
    virtual sc_cor* create(size_t stack_size, sc_cor_fn* fn, void* arg) = 0;
    virtual void yield(sc_cor* next_cor) = 0;
    virtual void abort(sc_cor* next_cor) = 0;
    virtual sc_cor* get_main() = 0;
};
```

### 7.2 平台實作

| 檔案 | 實作方式 | 適用平台 |
|------|---------|---------|
| sc_cor_pthread.cpp | POSIX 執行緒 | Linux/Unix |
| sc_cor_qt.cpp | QuickThreads | 通用 |
| sc_cor_fiber.cpp | Windows Fiber | Windows |
| sc_cor_std_thread.cpp | C++ std::thread | 現代 C++ |

---

## 8. sc_module - 模組機制

### 8.1 類別結構

```cpp
class sc_module : public sc_object_host, public sc_process_host {
    sc_sensitive     sensitive;
    sc_sensitive_pos sensitive_pos;
    sc_sensitive_neg sensitive_neg;
    
    void before_end_of_elaboration();
    void end_of_elaboration();
    void start_of_simulation();
    void end_of_simulation();
};
```

### 8.2 行程宣告宏

```cpp
SC_METHOD(func)     // 宣告方法行程
SC_THREAD(func)       // 宣告執行緒行程
SC_CTHREAD(func, edge) // 宣告時脈執行緒行程
```

### 8.3 RTL 對應

| SystemC | Verilog |
|---------|---------|
| sc_module | module |
| SC_METHOD | always_comb |
| SC_THREAD | always |
| SC_CTHREAD | always @(posedge clk) |

---

## 9. sc_time - 時間機制

### 9.1 時間單位

```cpp
enum sc_time_unit {
    SC_FS = 0,   // 飛秒 10^-15
    SC_PS = 1,   // 皮秒 10^-12
    SC_NS = 2,   // 奈秒 10^-9
    SC_US = 3,   // 微秒 10^-6
    SC_MS = 4,   // 毫秒 10^-3
    SC_SEC = 5   // 秒
};
```

### 9.2 時間解析度

```cpp
sc_set_time_resolution(1, SC_PS);  // 設定解析度為 1ps
sc_time t(10, SC_NS);               // 10ns = 10000ps
```

---

## 總結

SystemC Kernel 是整個模擬器的核心，主要負責：

1. **sc_simcontext** - 模擬器控制器，管理所有元件
2. **sc_process_b** - 行程基礎，定義三種行程類型
3. **sc_method_process** - 組合邏輯行程
4. **sc_thread_process** - 時序邏輯行程
5. **sc_event** - 事件通知機制
6. **sc_cor** - 協程實現暫停/恢復
7. **sc_module** - 硬體模組抽象
8. **sc_time** - 時間管理

理解這些核心機制是掌握 SystemC 的基礎！