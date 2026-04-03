# sc_fifo_ports.h - FIFO 專用埠類別

## 概觀

這個檔案定義了 `sc_fifo_in<T>` 和 `sc_fifo_out<T>` 兩個埠類別，分別用來連接 FIFO 通道的讀取端和寫入端。它們封裝了 `sc_port`，提供更方便的存取方法，讓模組可以直接呼叫 `read()` / `write()` 而不需要透過 `->` 運算子。

## 核心概念 / 生活化比喻

### 水管接頭

想像兩條水管要連接到一個水塔（FIFO）：

- **進水管接頭** (`sc_fifo_in`)：只能從水塔取水，連接到水塔的出水口
- **出水管接頭** (`sc_fifo_out`)：只能往水塔注水，連接到水塔的入水口
- 接頭的規格（介面）決定了它只能接到 FIFO 這種水塔，不能接到水龍頭（普通 signal）

```mermaid
graph LR
    Producer["生產者模組"] -->|sc_fifo_out| FIFO["sc_fifo 通道"]
    FIFO -->|sc_fifo_in| Consumer["消費者模組"]
```

## 類別詳細說明

### `sc_fifo_in<T>` - FIFO 輸入埠

```cpp
template <class T>
class sc_fifo_in
: public sc_port<sc_fifo_in_if<T>, 0, SC_ONE_OR_MORE_BOUND>
```

#### 型別定義

| 型別別名 | 實際型別 | 說明 |
|----------|---------|------|
| `data_type` | `T` | 資料型別 |
| `if_type` | `sc_fifo_in_if<T>` | 所綁定的介面型別 |
| `base_type` | `sc_port<...>` | 基礎埠型別 |

#### 快捷方法

這些方法直接轉發給底層介面，免去 `(*this)->` 的寫法：

| 方法 | 說明 |
|------|------|
| `read(data_type&)` | 阻塞讀取 |
| `read()` | 阻塞讀取（回傳值） |
| `nb_read(data_type&)` | 非阻塞讀取 |
| `num_available()` | 可讀樣本數 |
| `data_written_event()` | 資料寫入事件 |

#### 事件尋找器

```cpp
sc_event_finder& data_written() const;
```

用於**靜態敏感度**設定。例如：

```cpp
SC_METHOD(process);
sensitive << fifo_in.data_written();
```

這樣當 FIFO 有新資料寫入時，`process` 就會被觸發。內部使用 `sc_event_finder::cached_create` 快取建立事件尋找器。

### `sc_fifo_out<T>` - FIFO 輸出埠

```cpp
template <class T>
class sc_fifo_out
: public sc_port<sc_fifo_out_if<T>, 0, SC_ONE_OR_MORE_BOUND>
```

結構與 `sc_fifo_in` 對稱：

| 方法 | 說明 |
|------|------|
| `write(const data_type&)` | 阻塞寫入 |
| `nb_write(const data_type&)` | 非阻塞寫入 |
| `num_free()` | 可寫空位數 |
| `data_read_event()` | 資料被讀走事件 |
| `data_read()` | 靜態敏感度用的事件尋找器 |

## 建構子一覽

兩個類別都提供多種建構子，支援不同的建立方式：

| 建構方式 | 說明 |
|----------|------|
| 預設建構 | 自動命名 |
| `const char* name_` | 指定名稱 |
| `in_if_type& interface_` | 直接綁定介面 |
| `in_port_type& parent_` | 綁定到父埠 |
| `this_type& parent_` | 綁定到同類型父埠 |
| 名稱 + 介面/父埠 組合 | 以上的具名版本 |

## 設計原理

### 為何不直接用 `sc_port`？

1. **便利性**：`fifo_in.read()` 比 `fifo_in->read()` 更直觀
2. **靜態敏感度支援**：`data_written()` / `data_read()` 事件尋找器是 FIFO 特有的
3. **型別安全**：確保只能連接到 FIFO 介面

### `SC_ONE_OR_MORE_BOUND` 綁定策略

這表示埠必須至少綁定一個通道。雖然 `sc_port` 的第二個模板參數設為 0（表示無上限），但搭配 `SC_ONE_OR_MORE_BOUND` 確保不會有未綁定的埠。

### 事件尋找器的快取

`m_written_finder_p` / `m_read_finder_p` 使用 `mutable` 修飾，因為建立事件尋找器是延遲的（lazy），在 `const` 方法中也需要修改這個指標。解構子負責刪除。

## 相關檔案

- `sc_fifo.h` - FIFO 通道實作
- `sc_fifo_ifs.h` - FIFO 介面定義
- `sc_port.h` - 通用埠基礎類別
- `sc_signal_ports.h` - 信號埠（類似但用於 signal 通道）
