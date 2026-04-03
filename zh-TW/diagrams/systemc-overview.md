# SystemC 架構總覽圖

> 本頁面提供 SystemC 框架各子系統之間的全局關聯圖。

## 子系統關聯圖

```mermaid
flowchart TB
    subgraph 使用者層
        USER[使用者模型<br/>sc_module 子類別]
        MAIN[sc_main<br/>程式入口]
    end

    subgraph 核心引擎["sysc/kernel/"]
        SIM[sc_simcontext<br/>模擬上下文]
        EVT[sc_event<br/>事件系統]
        PROC[sc_process_b<br/>process 管理]
        TIME[sc_time<br/>時間管理]
        MOD[sc_module<br/>模組系統]
        OBJ[sc_object<br/>物件基礎]
        RUN[sc_runnable<br/>排程佇列]
        COR[sc_cor<br/>協程支援]
    end

    subgraph 通訊層["sysc/communication/"]
        SIG[sc_signal<br/>信號通道]
        PORT[sc_port<br/>端口系統]
        IF[sc_interface<br/>介面]
        PRIM[sc_prim_channel<br/>基礎通道]
        FIFO[sc_fifo<br/>FIFO 通道]
        MUT[sc_mutex / sc_semaphore<br/>同步原語]
    end

    subgraph 資料型別["sysc/datatypes/"]
        BIT[bit/<br/>sc_logic, sc_bv, sc_lv]
        INT[int/<br/>sc_int, sc_uint, sc_bigint]
        FX[fx/<br/>sc_fixed, sc_fix]
    end

    subgraph 追蹤["sysc/tracing/"]
        TRC[sc_trace_file<br/>追蹤基礎]
        VCD[sc_vcd_trace<br/>VCD 格式]
        WIF[sc_wif_trace<br/>WIF 格式]
    end

    subgraph TLM["tlm_core/ + tlm_utils/"]
        GP[tlm_generic_payload<br/>通用負載]
        SOC[tlm_initiator_socket<br/>tlm_target_socket]
        PEQ[peq_with_cb_and_phase<br/>事件佇列]
        QK[tlm_quantumkeeper<br/>時間量子]
    end

    subgraph 工具["sysc/utils/"]
        RPT[sc_report<br/>錯誤報告]
        VEC[sc_vector<br/>命名向量]
        MEM[sc_mempool<br/>記憶體池]
    end

    subgraph 平台["sysc/packages/"]
        QT[QuickThreads<br/>協程實作]
    end

    %% User → Core
    MAIN --> SIM
    USER --> MOD

    %% Core internal
    SIM --> EVT
    SIM --> PROC
    SIM --> TIME
    SIM --> RUN
    MOD --> OBJ
    MOD --> PROC
    PROC --> COR
    PROC --> EVT
    RUN --> PROC

    %% Communication → Core
    PORT --> IF
    SIG --> PRIM
    PRIM --> SIM
    FIFO --> PRIM
    MUT --> PRIM

    %% User → Communication
    USER --> PORT
    USER --> SIG

    %% Datatypes used everywhere
    SIG -.-> BIT
    SIG -.-> INT
    USER -.-> FX

    %% Tracing
    TRC --> SIM
    VCD --> TRC
    WIF --> TRC
    SIG -.-> TRC

    %% TLM
    SOC --> PORT
    GP -.-> INT
    PEQ --> EVT
    QK --> TIME

    %% Utils
    RPT --> SIM
    COR --> QT

    %% Styling
    style 核心引擎 fill:#e1f5fe
    style 通訊層 fill:#f3e5f5
    style 資料型別 fill:#e8f5e9
    style 追蹤 fill:#fff3e0
    style TLM fill:#fce4ec
    style 工具 fill:#f5f5f5
    style 平台 fill:#f5f5f5
```

## 依賴方向總覽

```mermaid
flowchart LR
    subgraph 基礎層
        utils[sysc/utils]
        packages[sysc/packages]
    end

    subgraph 核心層
        kernel[sysc/kernel]
        datatypes[sysc/datatypes]
    end

    subgraph 功能層
        comm[sysc/communication]
        tracing[sysc/tracing]
    end

    subgraph 抽象層
        tlm_core[tlm_core]
        tlm_utils[tlm_utils]
    end

    packages --> kernel
    utils --> kernel
    kernel --> comm
    kernel --> tracing
    datatypes --> comm
    comm --> tlm_core
    kernel --> tlm_core
    tlm_core --> tlm_utils

    style 基礎層 fill:#f5f5f5
    style 核心層 fill:#e1f5fe
    style 功能層 fill:#f3e5f5
    style 抽象層 fill:#fce4ec
```

## 文件統計

| 子系統 | 文件數 | 說明 |
|--------|--------|------|
| sysc/kernel | 42 | 模擬核心引擎 |
| sysc/communication | 28 | 通訊元件 |
| sysc/datatypes | 52 | 資料型別（bit + fx + int + misc） |
| sysc/tracing | 5 | 波形追蹤 |
| sysc/utils | 16 | 工具函式庫 |
| sysc/packages | 2 | QuickThreads 協程 |
| tlm_core | 18 | TLM 1.0 + 2.0 核心 |
| tlm_utils | 11 | TLM 工具套件 |
| topdown | 10 | Top-down 概念文件 |
| **合計** | **~195** | **含索引頁面** |
