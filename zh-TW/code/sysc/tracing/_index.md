# sysc/tracing/ - 波形追蹤子系統

> SystemC 模擬過程中的訊號記錄機制，將訊號值隨時間的變化匯出為 VCD 或 WIF 格式檔案，供後續波形檢視工具分析。

## 日常生活比喻

想像你在觀察一場足球比賽。**追蹤（tracing）** 就像賽事的「文字轉播」——每個時刻都記錄下球員（訊號）的位置和狀態。賽後，你可以用這份紀錄還原任何時刻發生了什麼。

- **sc_trace_file** = 轉播記者（抽象角色，負責「記錄」這件事）
- **sc_trace_file_base** = 記者的標準作業流程（開檔、時間校準、何時該記錄）
- **vcd_trace_file** = 用「VCD 格式」寫紀錄的記者
- **wif_trace_file** = 用「WIF 格式」寫紀錄的記者
- **sc_trace()** = 告訴記者「請幫我盯這個球員」

## 子系統概覽

```mermaid
flowchart TD
    subgraph 使用者介面
        API[sc_trace.h<br/>sc_trace 全域函式]
    end

    subgraph 基礎設施
        BASE_H[sc_trace_file_base.h<br/>共用追蹤基底類別]
        BASE_C[sc_trace_file_base.cpp<br/>時間刻度管理 / 生命週期]
        IDS[sc_tracing_ids.h<br/>錯誤訊息 ID]
    end

    subgraph 格式實作
        VCD_H[sc_vcd_trace.h]
        VCD_C[sc_vcd_trace.cpp<br/>VCD 格式輸出]
        WIF_H[sc_wif_trace.h]
        WIF_C[sc_wif_trace.cpp<br/>WIF 格式輸出]
    end

    API --> BASE_H
    BASE_H --> IDS
    VCD_H --> BASE_H
    WIF_H --> BASE_H
    VCD_C --> VCD_H
    WIF_C --> WIF_H
```

## 類別繼承階層

```mermaid
classDiagram
    sc_trace_file <|-- sc_trace_file_base
    sc_trace_file_base <|-- vcd_trace_file
    sc_trace_file_base <|-- wif_trace_file
    sc_stage_callback_if <|.. sc_trace_file_base

    class sc_trace_file {
        <<abstract>>
        +trace(object, name)*
        +write_comment(comment)*
        +set_time_unit(v, tu)*
        +delta_cycles(flag)
        #cycle(delta_cycle)*
        #event_trigger_stamp(event)
    }

    class sc_trace_file_base {
        #fp : FILE*
        #trace_unit_fs : unit_type
        #kernel_unit_fs : unit_type
        +filename()
        +delta_cycles()
        +set_time_unit(v, tu)
        #initialize()
        #open_fp()
        #do_initialize()*
        #add_trace_check(name)
        #timestamp_in_trace_units(high, low)
        -stage_callback(stage)
    }

    class vcd_trace_file {
        +traces : vector~vcd_trace*~
        +obtain_name()
        #do_initialize()
        #cycle(delta_cycle)
        -vcd_name_index : unsigned
    }

    class wif_trace_file {
        +traces : vector~wif_trace*~
        +obtain_name()
        #do_initialize()
        #cycle(delta_cycle)
        -wif_name_index : unsigned
    }
```

## VCD 內部追蹤物件階層

```mermaid
classDiagram
    vcd_trace <|-- vcd_T_trace~T~
    class vcd_trace {
        <<abstract>>
        +name : string
        +vcd_name : string
        +vcd_var_type : vcd_enum
        +bit_width : int
        +write(f)*
        +changed()*
        +print_variable_declaration_line(f, name)
        +print_data_line(f, rawdata)
        +strip_leading_bits(buf)$
    }
    class vcd_T_trace~T~ {
        -object : const T&
        -old_value : T
        +write(f)
        +changed() bool
    }
```

## 追蹤流程

```mermaid
sequenceDiagram
    participant User as 使用者程式碼
    participant API as sc_trace()
    participant TF as vcd_trace_file
    participant Kernel as sc_simcontext

    Note over User,Kernel: === 設定階段（模擬開始前）===
    User->>API: sc_create_vcd_trace_file("dump")
    API->>TF: new vcd_trace_file("dump")
    User->>API: sc_trace(tf, signal, "clk")
    API->>TF: trace(signal, "clk")
    TF->>TF: traces.push_back(new vcd_T_trace)

    Note over User,Kernel: === 模擬階段 ===
    Kernel->>TF: stage_callback(SC_PRE_TIMESTEP)
    TF->>TF: cycle(false)
    TF->>TF: initialize() (first time only)
    loop for each trace
        TF->>TF: changed()?
        TF->>TF: write(fp)
    end

    Note over User,Kernel: === 結束階段 ===
    User->>API: sc_close_vcd_trace_file(tf)
    API->>TF: delete tf
    TF->>TF: fclose(fp)
```

## 檔案清單

| 檔案 | 說明 |
|------|------|
| [sc_trace.md](sc_trace.md) | 追蹤公用 API：`sc_trace_file` 抽象類別與全域 `sc_trace()` 函式 |
| [sc_trace_file_base.md](sc_trace_file_base.md) | 追蹤檔案共用基底類別：時間刻度、檔案生命週期、callback 機制 |
| [sc_tracing_ids.md](sc_tracing_ids.md) | 追蹤子系統錯誤/警告訊息 ID 定義 |
| [sc_vcd_trace.md](sc_vcd_trace.md) | VCD（Value Change Dump）格式追蹤實作 |
| [sc_wif_trace.md](sc_wif_trace.md) | WIF（Waveform Interchange Format）格式追蹤實作 |

## 相關子系統

- `sysc/kernel/` — 模擬核心，提供 `sc_simcontext`、`sc_event`、`sc_time`
- `sysc/communication/` — 訊號介面 `sc_signal_in_if`，追蹤函式可直接追蹤訊號
- `sysc/datatypes/` — 各種資料型別（`sc_logic`, `sc_bv_base` 等），追蹤需支援所有型別
- `sysc/utils/` — 錯誤回報機制 `sc_report`
