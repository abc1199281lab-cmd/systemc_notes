# SystemC Entry Point 分析

> **檔案**:
> - `ref/systemc/src/sysc/kernel/sc_main.cpp`
> - `ref/systemc/src/sysc/kernel/sc_main_main.cpp`

## 1. 概述
SystemC 應用程式的進入點 (entry point) 是獨特的。不像標準 C++ 應用程式由使用者定義 `main()` 作為進入點，SystemC 函式庫自己定義了 `main()`。使用者必須改為定義 `sc_main()`。

這兩個檔案充當 "包裝器 (wrapper)" 或 "墊片 (shim)"，將標準 C++ `main()` 連接到 SystemC 核心的初始化，最後再連接到使用者的 `sc_main()`。

---

## 2. 程式碼分析

### 2.1 `sc_main.cpp` - 真實的進入點
這個檔案包含實際的 C++ `main` 函式。

```cpp
// sysc/kernel/sc_main.cpp
int main( int argc, char* argv[] )
{
    // 將執行轉發給 sc_main_main.cpp 中定義的 sc_elab_and_sim
    return sc_core::sc_elab_and_sim( argc, argv );
}
```

- **角色**: 它只是捕捉命令列參數，並立即將控制權委派給 `sc_elab_and_sim`。
- **設計模式**: **Proxy/Wrapper**。它攔截程式啟動，以確保 SystemC 在使用者程式碼執行之前正確初始化。

### 2.2 `sc_main_main.cpp` - 執行協調者
這個檔案包含設定 SystemC 環境的核心邏輯。

#### 關鍵函式: `sc_elab_and_sim`

```cpp
// sysc/kernel/sc_main_main.cpp
int sc_elab_and_sim( int argc, char* argv[] )
{
    // ... (參數處理) ...

    try
    {
        pln(); // 列印函式庫名稱/版權

        // 1. Initialization Marker (初始化標記)
        sc_in_action = true; // 全域旗標，表示模擬器正在活動

        // 2. Call User's sc_main (呼叫使用者的 sc_main)
        // 注意: argv 被複製以防止使用者程式碼修改影響內部狀態
        status = sc_main( argc, &argv_call[0] );

        // 3. Cleanup (清理)
        sc_in_action = false;
    }
    catch( ... )
    {
        // ... (透過 sc_report_handler 處理例外) ...
    }

    // ... (廢棄警告檢查) ...

    return status;
}
```

- **安全機制**: 它將 `sc_main` 包裝在 `try-catch` 區塊中，以確保例外 (如 `sc_report`) 被 SystemC 報告處理器優雅地捕捉和處理，而不是讓程式崩潰。
- **狀態管理**: 使用 `sc_in_action` 簡單的全域旗標來追蹤特定的 shim 是否正在執行。
- **Argv 保護**: 在傳遞給 `sc_main` 之前，它對 `argv` 進行深層複製 (deep copy)。這是防禦性程式設計，以保護原始的命令列參數。

---

## 3. 硬體 / RTL 對應
雖然這是純軟體鷹架 (scaffolding)，但它對應到 HDL 中的 **Testbench Top**。

| SystemC | Verilog / SystemVerilog |
| :--- | :--- |
| `main()` (函式庫定義) | Simulator Kernel (例如 `vsim`, `irun`) |
| `sc_main()` (使用者定義) | `module top;` 或 `program automatic test;` |
| `sc_elab_and_sim` | 編譯和載入設計的過程 (`elaboration`) |

- **概念**: 就像你不會寫 Verilog 模擬器的 "開機程式碼" 一樣，你也不會在 SystemC 中寫 `main()`。你寫最上層的 testbench (`sc_main`)，而核心處理開機順序。

---

## 4. 基於程式碼的關鍵重點
1.  **不要定義 `main()`**: 如果你嘗試在 SystemC 應用程式中定義自己的 `main` 函式，你會得到連結器錯誤 (multiple definitions of `main`)。
2.  **例外處理**: 核心被設計來捕捉從你的 `sc_main` 丟出的例外。
3.  **函式庫初始化**: `sc_elab_and_sim` 技術上是 "主迴圈" 的包裝器，雖然實際的模擬迴圈發生在 `sc_start()` 內部，而 `sc_start()` 是在你的 `sc_main()` *內部* 被呼叫的。
