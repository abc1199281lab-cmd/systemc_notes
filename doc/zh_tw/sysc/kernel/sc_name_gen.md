# SystemC Kernel: sc_name_gen

> **Source**: `ref/systemc/src/sysc/kernel/sc_name_gen.cpp`

## 1. 概述
`sc_name_gen` 是負責確保所有 SystemC 物件擁有唯一名稱的工具。如果使用者提供重複的名稱 (或沒有名稱)，這個類別會產生一個唯一的後綴。

## 2. 機制

### 2.1 對應表 (The Map)
它維護一個雜湊表 `m_unique_name_map`:
-   **Key**: `const char*` (使用者提供的基本名稱)。
-   **Value**: `int*` (此名稱出現次數的計數器)。

### 2.2 產生邏輯 (`gen_unique_name`)
當被要求為 `basename` 產生唯一名稱時：
1.  **第一次出現**:
    -   如果 `preserve_first` 為 true (訊號/模組的預設值)：返回的名稱就是 `basename`。
    -   計數器初始化為 0。
2.  **後續出現** (衝突):
    -   計數器遞增。
    -   新名稱變成 `basename_N` (例如 `my_signal_0`, `my_signal_1`)。

## 3. 用法
-   由 `sc_object_manager` 在建立實例時使用。
-   確保即使使用者在迴圈中執行 `sc_signal<int> sig("val");`，每個訊號在內部也會獲得唯一的階層名稱 (雖然明確命名較佳)。

---

## 4. 關鍵重點
-   **唯一性**: 保證在相同範圍內 (通常是兄弟物件) 的名稱唯一性。
-   **確定性**: 只要物件建立順序是確定性的，產生的名稱就會是確定性的。
