# TLM 實例特定擴充 (Instance Specific Extensions) 分析

> **檔案**: `ref/systemc/src/tlm_utils/instance_specific_extensions.h`

## 1. 概述
實例特定擴充 (Instance Specific Extensions, ISPE) 允許模組將資料附加到交易上，且這些資料對該模組實例是 **私有的**。標準的 TLM 擴充是全域的；如果模組 A 附加了一個擴充，模組 B 也能看到它。ISPE 解決了這種可見性/隱私問題。

## 2. 機制
- **載體 (Carrier)**: 一個稱為 `instance_specific_extension_carrier` 的特殊擴充會附加到 generic payload 上。這個載體持有一個私有擴充的容器。
- **存取器 (Accessor)**: 每個想要使用 ISPE 的模組實例必須有一個 `instance_specific_extension_accessor` 成員。
- **存取**:
  ```cpp
  // 設定
  m_accessor(trans).set_extension(my_ext);
  
  // 獲取
  MyExt* ext = m_accessor(trans).get_extension<MyExt>();
  ```

## 3. 運作方式
1.  當呼叫 `m_accessor(trans)` 時，它會檢查交易是否已經擁有載體。
2.  如果沒有，它會建立一個並將其附加。
3.  然後它會回傳一個輔助物件 (`instance_specific_extensions_per_accessor`)，該物件使用存取器的唯一 ID 在容器中進行索引。
4.  這確保了模組 A 的存取器查找的是槽位 A，而模組 B 的存取器查找的是槽位 B，即使它們附加到同一個交易上。

## 4. 使用場景
- **路由器/合併器 (Routers/Mergers)**: 路由器在轉發交易時可能需要附加路由資訊 (例如 "這來自港口 3")。它不希望其他組件依賴或檢查這個內部實作細節。
- **遺留橋接器 (Legacy Bridges)**: 在不同匯流排標準之間進行橋接時，儲存原始協定資訊。
