# TLM 分析港口 (Analysis Port) 分析

> **檔案**: `ref/systemc/src/tlm_core/tlm_1/tlm_analysis/tlm_analysis_port.h`

## 1. 概述
`tlm_analysis_port` 提供了一種交易的廣播機制。與通常為 1 對 1 的標準發起者/目標港口不同，分析港口是 **1 對多** 的。

## 2. 實作
- **容器**: 使用 `std::deque<tlm_analysis_if<T>*>` 來儲存所有已綁定介面 (訂閱者) 的指標。
- **綁定**: `bind()` 將介面加到清單中。
- **解綁**: `unbind()` 從清單中移除介面。
- **寫入**: `write(const T& t)` 方法遍歷清單並在每個綁定的訂閱者上呼叫 `write(t)`。

## 3. 用法
它主要用於需要觀察流量而不干擾交易流的 **被動組件 (passive components)**：
- **計分板 (Scoreboards)**: 用於檢查正確性。
- **覆蓋率收集器 (Coverage Collectors)**: 用於追蹤哪些地址/模式已被測試過。
- **記錄器/監視器 (Loggers/Monitors)**: 用於列印追蹤資訊。

## 4. 關鍵重點
1.  **觀察者模式 (Observer Pattern)**: 它實作了觀察者模式。
2.  **零時間**: 分析港口通常用於零時間、非阻塞式的通訊 (呼叫會立即完成)。
