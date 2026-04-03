# SystemC 官方範例 - 全域架構總覽

## 全域架構圖

```mermaid
flowchart TB
    subgraph SYSC["SystemC 核心範例 (sysc/)"]
        direction TB

        subgraph BASIC["基礎範例"]
            SF["simple_fifo<br/>生產者-消費者"]
            RSA["rsa<br/>大數運算"]
            SP["simple_perf<br/>效能建模"]
        end

        subgraph DSP["訊號處理"]
            PIPE["pipe<br/>3 級管線"]
            FIR["fir<br/>FIR 濾波器<br/>(行為+RTL)"]
            FFT["fft<br/>FFT<br/>(浮點+定點)"]
        end

        subgraph ARCH["系統架構"]
            PKT["pkt_switch<br/>封包交換"]
            BUS["simple_bus<br/>匯流排仲裁"]
            CPU["risc_cpu<br/>RISC CPU"]
        end

        subgraph VER["版本特性"]
            V21["2.1<br/>動態 process<br/>fork-join<br/>barrier"]
            V23["2.3<br/>握手協定<br/>async event"]
            V24["2.4<br/>in-class init"]
            ASYNC["async_suspend<br/>跨執行緒事件"]
        end
    end

    subgraph TLM["TLM 交易層模型範例 (tlm/)"]
        direction TB

        COMMON["common/<br/>共用元件庫<br/>(initiator, target,<br/>memory, bus)"]

        subgraph LT_GROUP["Loosely-Timed (同步)"]
            LT["lt<br/>基本 LT"]
            LT_DMI["lt_dmi<br/>直接記憶體"]
            LT_TD["lt_temporal_decouple<br/>時間解耦"]
            LT_ME["lt_mixed_endian<br/>混合位元序"]
            LT_EXT["lt_extension_mandatory<br/>必要擴展"]
        end

        subgraph AT_GROUP["Approximately-Timed (非同步)"]
            AT1["at_1_phase<br/>單階段"]
            AT2["at_2_phase<br/>雙階段"]
            AT4["at_4_phase<br/>四階段"]
            AT_EO["at_extension_optional<br/>可選擴展"]
            AT_MT["at_mixed_targets<br/>混合目標"]
            AT_OOO["at_ooo<br/>亂序完成"]
        end

        COMMON --> LT_GROUP
        COMMON --> AT_GROUP
    end

    BASIC --> DSP
    BASIC --> ARCH
    DSP --> ARCH
    SYSC --> TLM

    style BASIC fill:#d4edda
    style DSP fill:#fff3cd
    style ARCH fill:#f8d7da
    style VER fill:#cce5ff
    style LT_GROUP fill:#e2d9f3
    style AT_GROUP fill:#fce4ec
```

## 依賴方向總覽

```mermaid
flowchart LR
    subgraph 學習路徑
        direction LR
        A["入門概念<br/>simple_fifo<br/>pipe"] --> B["進階建模<br/>fir, fft<br/>pkt_switch"]
        B --> C["系統設計<br/>simple_bus<br/>risc_cpu"]
        A --> D["TLM 基礎<br/>common<br/>lt"]
        D --> E["TLM 進階<br/>lt_dmi<br/>at_2_phase<br/>at_4_phase"]
        E --> F["TLM 專精<br/>at_ooo<br/>extensions"]
    end
```

## 檔案統計

| 分類 | 範例數 | 原始碼檔案 | 文件檔案 | 含 spec.md |
| --- | --- | --- | --- | --- |
| sysc 基礎 | 3 | 4 | 11 | 2 |
| sysc 管線/DSP | 3 | 39 | 25 | 3 |
| sysc 系統架構 | 3 | 58 | 29 | 3 |
| sysc 版本特性 | 4 | 19 | 19 | 0 |
| tlm common | 1 | 40 | 9 | 1 |
| tlm LT | 5 | 28 | 10 | 0 |
| tlm AT | 6 | 35 | 12 | 0 |
| topdown | - | - | 6 | - |
| **合計** | **25** | **226** | **121** | **9** |

## 文件類型說明

| 類型 | 說明 | 範例 |
| --- | --- | --- |
| `_index.md` | 子系統總覽頁，含架構圖和檔案列表 | `code/sysc/pipe/_index.md` |
| `*.md` (per-file) | 對應原始碼檔案的詳細解析 | `code/sysc/pipe/stage1.md` |
| `spec.md` | 硬體 IP 規格說明（用軟體類比） | `code/sysc/pipe/spec.md` |
| topdown `*.md` | 跨範例的概念性文件 | `topdown/tlm-explained.md` |

## 軟體類比速查表

| SystemC / 硬體概念 | 軟體類比 |
| --- | --- |
| FIFO | Python queue.Queue |
| Pipeline | Unix pipe / ETL chain |
| FIR Filter | 滑動視窗加權平均 |
| FFT | 頻譜分析器 / 音樂等化器 |
| Packet Switch | 網路路由器 / RabbitMQ |
| Bus Arbitration | 共享資源 + Mutex / 執行緒排程器 |
| RISC CPU | 指令解譯器 + 快取系統 |
| TLM LT | 同步 HTTP (fetch + await) |
| TLM AT | 非同步 HTTP (callback/asyncio.Future) |
| TLM DMI | mmap / kernel bypass |
| TLM Extension | 自訂 HTTP Header |
| sc_module | class / component |
| sc_port | dependency injection |
| sc_signal | Observable / reactive variable |
| SC_THREAD | coroutine / Python coroutine (asyncio) |
| SC_METHOD | event callback |
| sc_event | condition variable / asyncio.Future |
| Delta cycle | microtask queue |
