# 2026-02-08-kernel-event

## 1. sc_event 概論
`sc_event` 是 SystemC 模擬核心中唯一的同步機制。它本身並不攜帶資料（Data），僅作為一個時間點的通知標記，用於觸發對該事件感興趣的行程。

## 2. 通知路徑 (Notification Paths)
`sc_event::notify()` 支援三種不同的通知路徑，這直接決定了排程器的行為：

### A. 立即通知 (Immediate Notification)
- **語法**：`e.notify();`（無參數）。
- **機制**：繞過排程器的隊列，立即將所有敏感於此事件的行程加入當前 Delta Cycle 的「可執行隊列」。
- **副作用**：可能導致在同一個 Evaluate 階段產生新的執行需求。

### B. Delta 通知 (Delta Notification)
- **語法**：`e.notify(SC_ZERO_TIME);`。
- **機制**：將事件通知排入排程器的「Delta 通知隊列」。
- **執行點**：在當前 Delta Cycle 的 Update 階段之後，下一個 Delta Cycle 的 Evaluate 階段之前觸發。

### C. 時間通知 (Timed Notification)
- **語法**：`e.notify(t);`（其中 $t > 0$）。
- **機制**：將事件通知排入排程器的「時間隊列」。
- **執行點**：當模擬時間推進到 $T_{now} + t$ 時觸發。

## 3. 動態敏感度 (Dynamic Sensitivity)
除了在建構子中使用 `sensitive << e;` 設定靜態敏感度外，SystemC 支援在執行期動態更改敏感度：
- **SC_METHOD**：使用 `next_trigger(e);`。當前執行結束後，下次觸發將由 `e` 決定。
- **SC_THREAD**：使用 `wait(e);`。行程在此掛起，直到 `e` 觸發後繼續執行。

## 4. 事件列表 (Event Lists)
支援邏輯組合（AND/OR）：
- **OR 組合**：`wait(e1 | e2);`（任一觸發即繼續）。
- **AND 組合**：`wait(e1 & e2);`（全部觸發才繼續）。

## 5. 原始碼參考
- `src/sysc/kernel/sc_event.h`: 類別定義與介面。
- `src/sysc/kernel/sc_event.cpp`: 通知邏輯與隊列管理。

---
*Source: ref/systemc/src/sysc/kernel/*
