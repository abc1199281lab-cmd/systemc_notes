# 2026-02-08-communication-fifo

## 1. sc_fifo 概論
`sc_fifo<T>` 是 SystemC 提供的同步 FIFO (First-In-First-Out) 通道，用於在兩個行程之間安全地傳遞資料流。它繼承自 `sc_prim_channel`，並實作了 `sc_fifo_in_if<T>` 與 `sc_fifo_out_if<T>` 雙向介面。

## 2. 阻塞式同步機制 (Blocking Synchronization)
`sc_fifo` 的核心特性是**阻塞式讀寫**，這與 `sc_signal` 的非阻塞行為形成鮮明對比：

### 阻塞讀取 (Blocking Read)
```cpp
void read(T& val) {
    while (num_available() == 0) {
        wait(m_data_written_event);  // FIFO 空時等待寫入事件
    }
    nb_read(val);  // 非阻塞讀取
}
```
當 FIFO 為空時，讀取行程會在 `wait()` 處掛起，直到有資料被寫入。

### 阻塞寫入 (Blocking Write)
```cpp
void write(const T& val) {
    while (num_free() == 0) {
        wait(m_data_read_event);  // FIFO 滿時等待讀取事件
    }
    nb_write(val);  // 非阻塞寫入
}
```
當 FIFO 滿時，寫入行程會等待讀取行程消耗資料。

### 行程間的握手
這種阻塞機制實現了**生產者-消費者模式**的同步：
- 生產者 (`write`) 與消費者 (`read`) 透過事件自動協調。
- 無需手動管理旗號或條件變量。

## 3. 非阻塞介面 (Non-Blocking Interface)
提供檢查式操作，適用於測試平台或需要超時處理的場景：
```cpp
bool nb_read(T& val);      // 若 FIFO 空，返回 false
bool nb_write(const T& val); // 若 FIFO 滿，返回 false

int num_available() const;  // 可讀取數量
int num_free() const;       // 可寫入空間
```

## 4. 內部緩衝區實作
### 環形緩衝區 (Circular Buffer)
```cpp
T*  m_buf;      // 動態分配的陣列
int m_ri;       // 讀取索引 (Read Index)
int m_wi;       // 寫入索引 (Write Index)
int m_size;     // FIFO 深度 (容量)
```
環形設計高效利用記憶體，避免資料搬移。

### Delta Cycle 書籤計數
```cpp
int m_num_readable;  // 本 Delta Cycle 可讀總量
int m_num_read;      // 本 Delta Cycle 已讀取量
int m_num_written;   // 本 Delta Cycle 已寫入量
```
這些計數器確保在同一 Delta Cycle 內的多次讀寫正確結算，並在 `update()` 時重置。

## 5. 事件通知機制
`sc_fifo` 維護兩個核心事件：
- **`m_data_written_event`**: 當有資料被寫入時觸發（喚醒等待讀取的行程）。
- **`m_data_read_event`**: 當有資料被讀取時觸發（喚醒等待寫入的行程）。

### Update Phase 處理
與 `sc_signal` 類似，`sc_fifo` 的 `update()` 方法在 Update Phase 執行：
1. 重置 Delta Cycle 計數器 (`m_num_read`, `m_num_written`)。
2. 根據實際變化通知 `data_written` 或 `data_read` 事件。

## 6. 單一讀寫者限制
`sc_fifo` 限制為**單一讀者、單一寫者**：
```cpp
void register_port(sc_port_base& port, const char* if_typename) {
    // 檢查：若已有 reader/writer，報告 SC_ID_MORE_THAN_ONE_FIFO_READER_ 錯誤
}
```
這確保了 FIFO 行為的確定性，避免多寫入者的競態條件。

## 7. 原始碼參考
- `src/sysc/communication/sc_fifo.h`: `sc_fifo<T>` 模板類別實作。
- `src/sysc/communication/sc_fifo_ifs.h`: 輸入/輸出介面定義。

---
*Source: ref/systemc/src/sysc/communication/*
