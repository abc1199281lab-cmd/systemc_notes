# SystemC 資料型別總覽 - datatypes 目錄

本目錄包含 SystemC 所有硬體模擬用的資料型別實作。這些型別是 SystemC 區別於一般 C++ 程式庫的核心：它們讓軟體工程師能夠在 C++ 中精確描述硬體電路中的數位訊號。

## 為什麼需要特殊的資料型別？

想像你在用樂高積木蓋房子。普通的 C++ 型別（`int`、`bool`）就像是固定大小的樂高磚塊——你只能用 8 位元、16 位元、32 位元或 64 位元。但在硬體設計中，你可能需要一個 13 位元的計數器，或是一條 128 位元的匯流排。SystemC 的資料型別就像是可以任意切割大小的樂高磚塊。

此外，真實的電路裡，一條線不只有 0 和 1 兩種狀態——它還可能處於「高阻抗」(Z) 或「未知」(X) 狀態。這就像交通號誌除了紅燈和綠燈之外，還可能故障（未知）或完全關閉（高阻抗）。

## 子目錄總覽

| 子目錄 | 用途 | 日常比喻 |
|--------|------|----------|
| `bit/` | 位元與邏輯向量型別 | 開關陣列——每個開關可以是開/關（二值）或開/關/斷線/未知（四值） |
| `int/` | 任意寬度整數型別 | 可調大小的數字計算器，支援有號/無號、任意位元寬度 |
| `fx/`  | 定點數型別 | 帶小數點的硬體計算器，用於 DSP 和需要精確小數運算的場合 |
| `misc/`| 雜項工具型別 | 輔助工具箱，包含數值表示法轉換等通用功能 |

## 型別之間的關係

```mermaid
graph TB
    subgraph "bit/ - 位元與邏輯型別"
        sc_bit["sc_bit<br/>(已棄用, 用 bool 取代)"]
        sc_logic["sc_logic<br/>(四值邏輯: 0/1/X/Z)"]
        sc_bv["sc_bv&lt;W&gt;<br/>(位元向量)"]
        sc_lv["sc_lv&lt;W&gt;<br/>(邏輯向量)"]
        sc_proxy["sc_proxy&lt;T&gt;<br/>(向量共用介面)"]
    end

    subgraph "int/ - 整數型別"
        sc_int["sc_int&lt;W&gt; / sc_uint&lt;W&gt;<br/>(固定寬度, ≤64位元)"]
        sc_bigint["sc_bigint&lt;W&gt; / sc_biguint&lt;W&gt;<br/>(任意寬度整數)"]
        sc_signed["sc_signed / sc_unsigned<br/>(動態寬度整數)"]
    end

    subgraph "fx/ - 定點數型別"
        sc_fixed["sc_fixed&lt;W,I&gt;<br/>(固定位元定點數)"]
        sc_fix["sc_fix<br/>(動態位元定點數)"]
    end

    subgraph "misc/ - 雜項"
        sc_value_base["sc_value_base<br/>(數值型別基底類別)"]
    end

    sc_bit --> sc_logic
    sc_logic --> sc_lv
    sc_bv --> sc_proxy
    sc_lv --> sc_proxy
    sc_bv -.->|"只含 0/1"| sc_lv
    sc_lv -.->|"可轉換"| sc_bigint
    sc_bv -.->|"可轉換"| sc_bigint
    sc_fixed -.->|"內部使用"| sc_signed
    sc_value_base -.->|"基底類別"| sc_signed
```

## 如何選擇合適的型別？

```mermaid
flowchart TD
    Start["我需要什麼型別？"] --> Q1{"需要表示<br/>X 或 Z 狀態嗎？"}
    Q1 -->|"是"| Q2{"單一位元<br/>還是多位元？"}
    Q1 -->|"否"| Q3{"需要做<br/>算術運算嗎？"}

    Q2 -->|"單一位元"| sc_logic_out["sc_logic"]
    Q2 -->|"多位元"| sc_lv_out["sc_lv&lt;W&gt;"]

    Q3 -->|"否"| Q4{"單一位元<br/>還是多位元？"}
    Q3 -->|"是"| Q5{"需要小數嗎？"}

    Q4 -->|"單一位元"| bool_out["bool"]
    Q4 -->|"多位元"| sc_bv_out["sc_bv&lt;W&gt;"]

    Q5 -->|"否"| Q6{"超過 64 位元嗎？"}
    Q5 -->|"是"| sc_fixed_out["sc_fixed / sc_fix"]

    Q6 -->|"否"| sc_int_out["sc_int / sc_uint"]
    Q6 -->|"是"| sc_bigint_out["sc_bigint / sc_biguint"]
```

## 相關檔案

- `bit/` - [位元與邏輯型別詳細文件](bit/_index.md)
- `int/` - 整數型別（sc_int, sc_uint, sc_bigint, sc_biguint, sc_signed, sc_unsigned）
- `fx/` - 定點數型別（sc_fixed, sc_ufixed, sc_fix, sc_ufix）
- `misc/` - 雜項工具（sc_value_base, sc_concatref）
