# at_ooo -- 原始碼詳解

> **原始碼路徑**: `ref/systemc/examples/tlm/at_ooo/`

## 軟體類比總覽

這個範例的核心概念可以用一個餐廳來比喻：

```
顧客 A 點了牛排（需要 30 分鐘）  --> 先點的
顧客 B 點了沙拉（需要 5 分鐘）   --> 後點的

In-order 餐廳:
  先做牛排 (30分鐘)，再做沙拉 (5分鐘) = 顧客 B 等了 35 分鐘

Out-of-order 餐廳:
  同時開始做，沙拉 5 分鐘就上桌 = 顧客 B 只等 5 分鐘
```

在 TLM 中，`at_target_ooo_2_phase` 就是這家 out-of-order 餐廳。

## 系統架構

```
example_system_top
  |-- SimpleBusAT<2, 2>       m_bus
  |-- at_target_2_phase       m_at_target_2_phase_1     (ID=201) -- 普通 target
  |-- at_target_ooo_2_phase   m_at_target_ooo_2_phase_1 (ID=202) -- OOO target
  |-- initiator_top           m_initiator_1              (ID=101)
  |-- initiator_top           m_initiator_2              (ID=102)
```

### OOO Target 的特殊延遲設定

```cpp
m_at_target_ooo_2_phase_1(
    "m_at_target_ooo_2_phase_1",
    202,
    "memory_socket_1",
    4*1024,                                // memory size
    4,                                     // memory width
    sc_core::sc_time(20, sc_core::SC_NS),  // accept_delay  (一般 target 是 10ns)
    sc_core::sc_time(100, sc_core::SC_NS), // read_delay    (一般 target 是 50ns)
    sc_core::sc_time(60, sc_core::SC_NS)   // write_delay   (一般 target 是 30ns)
);
```

OOO target 的延遲故意設得更長（accept: 20ns vs 10ns, read: 100ns vs 50ns），加上額外的 700ns OOO 延遲，這樣更容易觀察到亂序行為。

## OOO Target 實作：at_target_ooo_2_phase

### 標頭檔分析

`at_target_ooo_2_phase` 與 `at_target_2_phase` 幾乎相同，但多了兩個關鍵的成員變數：

```cpp
sc_core::sc_time    m_peq_delay_time;           // 目前的 PEQ 延遲
sc_core::sc_time    m_delay_for_out_of_order;   // 額外的 OOO 延遲 (700 ns)
```

### nb_transport_fw -- OOO 的核心機制

收到 `BEGIN_REQ` 時：

```cpp
m_target_memory.get_delay(gp, delay_time);  // 取得基本記憶體延遲
delay_time += m_accept_delay;
m_peq_delay_time = delay_time;

bool ooo_flag(false);
// 每隔一個請求，加上 700ns 的額外延遲
if (m_request_count++ % 2) {
    m_peq_delay_time += m_delay_for_out_of_order;  // +700ns!
    ooo_flag = true;
}

m_response_PEQ.notify(gp, m_peq_delay_time);  // 放入 PEQ
```

軟體類比：

```python
async def handle_request(request, request_number):
    base_delay = calculate_memory_delay(request)

    if request_number % 2 == 1:
        # 奇數請求：模擬慢速操作
        total_delay = base_delay + 700  # 額外 700ns
    else:
        # 偶數請求：正常速度
        total_delay = base_delay

    # 排入延遲佇列 -- 延遲短的會先被處理!
    await asyncio.sleep(total_delay)
    send_response(request)
```

### PEQ 排程造成的亂序

`peq_with_get` 是一個**按時間排序**的事件佇列。當兩個交易的到期時間不同時，先到期的會先被處理：

```
時間軸:
  t=0:  收到 Request A (奇數，delay = 120ns + 700ns = 820ns)
  t=5:  收到 Request B (偶數，delay = 120ns)

  t=125: Request B 到期 --> 送出 BEGIN_RESP for B  (B 先完成!)
  t=820: Request A 到期 --> 送出 BEGIN_RESP for A  (A 後完成!)
```

這就是 OOO 的本質：**PEQ 按到期時間排序，不是按放入時間排序**。

### begin_response_method

與 `at_target_2_phase` 完全相同：從 PEQ 取出到期的交易，執行記憶體操作，呼叫 `nb_transport_bw(BEGIN_RESP)`。差異不在這裡，而是在 `nb_transport_fw` 中不同的 PEQ 延遲造成的不同到期時間。

## 普通 Target vs OOO Target 比較

| 面向 | at_target_2_phase (ID=201) | at_target_ooo_2_phase (ID=202) |
| --- | --- | --- |
| 協定 | 2-phase | 2-phase |
| accept_delay | 10 ns | 20 ns |
| read_response_delay | 50 ns | 100 ns |
| write_response_delay | 30 ns | 60 ns |
| OOO 機制 | 無 | 奇數請求 +700ns 額外延遲 |
| 回應順序 | 基本上按請求順序 | 偶數請求可能比奇數請求先回應 |

## OOO 在真實系統中的應用

| 場景 | 軟體對應 | 為什麼需要 OOO |
| --- | --- | --- |
| CPU cache miss | `Promise.race([cacheHit, memoryAccess])` | Cache hit 應該立刻完成，不用等 miss 的 request |
| Multi-bank memory | 非同步 DB 查詢（多個 shard） | 不同 bank 的延遲不同 |
| DMA transfer | 並行下載多個檔案 | 小檔案應該先完成 |
| PCIe transaction | 並行 API 呼叫不同服務 | 快的服務不應該被慢的服務阻塞 |

## Initiator 端的 OOO 處理

`select_initiator` 天然支援 OOO，因為它使用 `waiting_bw_path_map` 來追蹤**每個交易的獨立狀態**。無論回應以什麼順序到達，它都能正確地：

1. 在 `nb_transport_bw` 中查找對應的交易
2. 更新該交易的狀態
3. 將完成的交易放入 `response_fifo`

軟體對應：

```python
# 每個請求有獨立的 Future，完成順序無關
pending = {}  # Dict[request_id, Future]

async def on_response(request_id, data):
    future = pending.pop(request_id)  # 找到對應的 Future
    future.set_result(data)            # 無論順序，都能正確完成
```

## 重點整理

| 概念 | 說明 |
| --- | --- |
| **OOO 核心** | 透過不同的 PEQ 延遲，讓某些交易比其他交易更晚完成 |
| **m_delay_for_out_of_order** | 700ns 的額外延遲，每隔一個請求加上 |
| **PEQ 排序** | `peq_with_get` 按到期時間排序，造成自然的亂序完成 |
| **Initiator 相容性** | `select_initiator` 用 per-transaction map 追蹤狀態，天然支援 OOO |
| **效能意義** | OOO 允許快速操作不被慢速操作阻塞，提升系統吞吐量 |
