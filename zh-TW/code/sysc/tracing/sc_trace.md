# sc_trace.h / sc_trace.cpp - 訊號追蹤公用 API

> 定義追蹤檔案的抽象介面 `sc_trace_file`，以及全域的 `sc_trace()` 函式群，讓使用者可以將各種資料型別的訊號記錄到追蹤檔案中。

## 日常生活比喻

想像你有一台「萬能攝影機」，不管你給它什麼東西（數字、布林值、邏輯訊號），它都能拍攝記錄。`sc_trace()` 就是這台攝影機的「拍攝按鈕」——你只要告訴它「拍哪個變數」和「取什麼名字」，它就會把該變數的變化全程記錄下來。

而 `sc_trace_file` 就是攝影機本身的「介面規格」——它只規定了攝影機必須具備哪些功能（記錄 bool、int、float...），但不管你內部用什麼底片（VCD 還是 WIF 格式）。

## 概覽

這個檔案是追蹤子系統的**公用入口**。它做兩件事：

1. **定義抽象基底類別 `sc_trace_file`**：宣告所有追蹤檔案必須實作的純虛擬函式
2. **提供全域 `sc_trace()` 函式多載**：為每種資料型別提供方便的呼叫介面

## sc_trace_file 抽象類別

### 類別定義

```cpp
class sc_trace_file
{
    friend class sc_simcontext;
public:
    sc_trace_file();

    // Pure virtual trace methods for each data type
    virtual void trace(const bool& object, const std::string& name) = 0;
    virtual void trace(const int& object, const std::string& name, int width) = 0;
    // ... (many more overloads)

    virtual void trace(const unsigned int& object,
                       const std::string& name,
                       const char** enum_literals) = 0;

    virtual void write_comment(const std::string& comment) = 0;
    virtual void space(int n);
    virtual void delta_cycles(bool flag);
    virtual void set_time_unit(double v, sc_time_unit tu) = 0;

protected:
    virtual void cycle(bool delta_cycle) = 0;
    const sc_dt::uint64& event_trigger_stamp(const sc_event& event) const;
    virtual ~sc_trace_file() {}
};
```

### 追蹤方法分類

追蹤方法透過巨集展開，分為兩類：

| 巨集 | 參數形式 | 適用型別 |
|------|---------|---------|
| `DECL_TRACE_METHOD_A(tp)` | `(const tp& object, const string& name)` | `bool`, `float`, `double`, `sc_logic`, `sc_signed` 等 |
| `DECL_TRACE_METHOD_B(tp)` | `(const tp& object, const string& name, int width)` | `char`, `short`, `int`, `long`, `int64`, `uint64` 等整數型別 |

**為什麼要分兩類？** 型別 A 是「寬度固定或自帶寬度資訊」的型別（如 `bool` 就是 1 bit，`sc_signed` 自己知道自己多寬）。型別 B 是「C++ 原生整數」，寬度可能因平台不同而異，所以需要額外指定 `width` 參數。

### 重要方法說明

| 方法 | 說明 |
|------|------|
| `trace(...)` | 註冊一個物件以供追蹤，每種型別都有對應的純虛擬版本 |
| `write_comment(comment)` | 在追蹤檔案中寫入一段註解 |
| `space(n)` | 設定欄位間距（大多數格式不使用，預設空實作） |
| `delta_cycles(flag)` | 是否追蹤 delta cycle 之間的變化 |
| `set_time_unit(v, tu)` | 設定追蹤檔案的時間單位 |
| `cycle(delta_cycle)` | [protected] 每次模擬推進時被呼叫，寫入該時刻的追蹤資料 |
| `event_trigger_stamp(event)` | [protected] 取得事件的觸發時間戳，供子類別追蹤事件用 |

## 全域 sc_trace() 函式

### 函式多載策略

全域 `sc_trace()` 函式也透過巨集展開，為每種型別提供兩個版本：

```cpp
// Reference version
void sc_trace(sc_trace_file* tf, const bool& object, const std::string& name);

// Pointer version
void sc_trace(sc_trace_file* tf, const bool* object, const std::string& name);
```

指標版本只是對引用版本的簡單包裝，會自動解引用。

### 實作邏輯

所有全域 `sc_trace()` 函式的實作都非常簡單——先檢查 `tf` 是否為空，然後轉呼叫 `tf->trace()`：

```cpp
void sc_trace(sc_trace_file* tf, const bool& object, const std::string& name)
{
    if (tf) {
        tf->trace(object, name);
    }
}
```

### 訊號介面模板特化

對於 `sc_signal_in_if<T>`，提供了模板版本，直接讀取訊號值來追蹤：

```cpp
template <class T>
void sc_trace(sc_trace_file* tf,
              const sc_signal_in_if<T>& object,
              const std::string& name)
{
    sc_trace(tf, object.read(), name);
}
```

同時為 `char`、`short`、`int`、`long` 的訊號介面提供了需要 `width` 參數的特化版本。

### 輔助函式

| 函式 | 說明 |
|------|------|
| `sc_trace_delta_cycles(tf, on)` | 開關 delta cycle 追蹤，內聯包裝 `tf->delta_cycles(on)` |
| `sc_write_comment(tf, comment)` | 寫入註解，內聯包裝 `tf->write_comment(comment)` |
| `tprintf(tf, format, ...)` | 類似 `printf` 的格式化寫入追蹤註解 |
| `sc_create_vcd_trace_file(name)` | 建立 VCD 格式追蹤檔案 |
| `sc_close_vcd_trace_file(tf)` | 關閉 VCD 追蹤檔案 |
| `sc_create_wif_trace_file(name)` | 建立 WIF 格式追蹤檔案 |
| `sc_close_wif_trace_file(tf)` | 關閉 WIF 追蹤檔案 |

### void* 的 Dummy 版本

```cpp
void sc_trace(sc_trace_file* tf, const void* object, const std::string& name);
```

這個版本是「萬用匹配」，當 C++ 編譯器找不到適合的多載版本時會選中它。它**不做任何事**，只發出一個 `SC_ID_TRACING_OBJECT_IGNORED_` 警告，告訴使用者「這種型別無法被追蹤」。

### 列舉追蹤（已棄用）

```cpp
void sc_trace(sc_trace_file* tf, const unsigned int& object,
              const std::string& name, const char** enum_literals);
```

這是 IEEE 1666 標準中已棄用的功能，用來追蹤列舉值並將字串字面值寫入追蹤檔案。第一次呼叫時會發出棄用警告。

## 設計決策

### 為什麼用巨集展開？

追蹤需要支援的型別數量龐大（`bool`、`char`、`short`、`int`、`long`、`int64`、`uint64`、`float`、`double`、`sc_bit`、`sc_logic`、`sc_signed`、`sc_unsigned` 等等），每種型別的追蹤函式簽名幾乎相同，只有型別名稱不同。用巨集展開可以避免大量重複程式碼，同時確保每種型別都有完整的宣告和定義。

### 為什麼 sc_trace_file 是 sc_simcontext 的 friend？

因為 `cycle()` 方法是 `protected` 的——只有模擬核心可以決定「什麼時候該記錄一筆資料」，使用者不應該手動呼叫 `cycle()`。

## 使用範例

```cpp
// Create trace file
sc_trace_file* tf = sc_create_vcd_trace_file("waveform");

// Set time unit
tf->set_time_unit(1, SC_NS);

// Register signals to trace
sc_signal<bool> clk;
sc_signal<int> data;
sc_trace(tf, clk, "clk");
sc_trace(tf, data, "data");

// After simulation
sc_close_vcd_trace_file(tf);
```

## 相關檔案

- [sc_trace_file_base.md](sc_trace_file_base.md) — `sc_trace_file` 的共用實作基底
- [sc_vcd_trace.md](sc_vcd_trace.md) — VCD 格式的具體實作
- [sc_wif_trace.md](sc_wif_trace.md) — WIF 格式的具體實作
- [sc_tracing_ids.md](sc_tracing_ids.md) — 追蹤相關錯誤訊息 ID
- `sysc/communication/sc_signal_ifs.h` — 訊號介面 `sc_signal_in_if<T>`
- `sysc/kernel/sc_event.h` — 事件類別，提供 `m_trigger_stamp`
