# SystemC FIFO 分析

> **檔案**: `ref/systemc/src/sysc/communication/sc_fifo.h`

## 1. 概述
`sc_fifo<T>` 是一個實作先進先出 (First-In-First-Out) 緩衝區的基本通道。它支援阻塞和非阻塞的讀取/寫入介面。

## 2. 實作

### 2.1 儲存 (Storage)
- **環形緩衝區 (Circular Buffer)**: 使用型別 `T` 和大小 `m_size` 的陣列 `m_buf`。
- **索引**: 邏輯使用 `m_ri` (讀取索引) 和 `m_wi` (寫入索引) 來管理環形包裝。

### 2.2 阻塞機制
- **事件**:
    - `m_data_read_event`: 當資料被讀取時通知 (意味著空間變得可用)。
    - `m_data_written_event`: 當資料被寫入時通知 (意味著資料變得可用)。
- **更新階段**:
    - `update()` 在 delta cycle 結束時被呼叫。
    - 它檢查 `m_num_read` 和 `m_num_written`。
    - 如果 `m_num_read > 0`，它通知 `m_data_read_event`。這喚醒任何阻塞在滿 FIFO 上的寫入者。
    - 如果 `m_num_written > 0`，它通知 `m_data_written_event`。這喚醒任何阻塞在空 FIFO 上的讀取者。

### 2.3 關鍵方法
- **`read(T&)`**: 如果 `num_available() == 0`，則阻塞在 `m_data_written_event` 上。
- **`write(const T&)`**: 如果 `num_free() == 0`，則阻塞在 `m_data_read_event` 上。
- **`nb_read` / `nb_write`**: 非阻塞版本，如果操作無法執行則立即返回 `false`。

---

## 3. 關鍵重點
1.  **確定性 (Deterministic)**: 請求-更新機制的使用保證了確定性。在 delta cycle `N` 寫入的值在 delta cycle `N+1` (有效地) 可被讀取，因為事件通知發生在更新階段。
