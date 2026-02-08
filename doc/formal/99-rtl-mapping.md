# 2026-02-08-rtl-mapping

## 1. RTL 映射概論
SystemC 與 RTL (Register Transfer Level) 描述語言 (Verilog/VHDL) 在語義上有本質對應關係。理解這種對應有助於將 SystemC 模型綜合成硬體，或將 Verilog/VHDL 程式碼轉換為 SystemC 進行高階驗證。

## 2. 組合邏輯 (Combinational Logic)

### Verilog `assign` vs SystemC `SC_METHOD`
```verilog
// Verilog
assign y = a & b | c;
```
```cpp
// SystemC
SC_METHOD(comb_proc);
sensitive << a << b << c;

void comb_proc() {
    y = a.read() & b.read() | c.read();
}
```
- **對應**: `assign` 語句 ↔ 對輸入敏感的 `SC_METHOD`。
- **Sematics**: 兩者都在輸入變化時立即重新計算輸出。

## 3. 時序邏邏 (Sequential Logic)

### Verilog `always @(posedge clk)` vs SystemC `SC_CTHREAD`
```verilog
// Verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) q <= 0;
    else q <= d;
end
```
```cpp
// SystemC
SC_CTHREAD(seq_proc, clk.pos());
reset_signal_is(rst_n, false);

void seq_proc() {
    q = 0;  // 同步重置
    while (true) {
        q = d.read();
        wait();  // 等待下一個時脈邊緣
    }
}
```
- **對應**: `always @(posedge clk)` ↔ 綁定時脈邊緣的 `SC_CTHREAD`。
- **實現**: SystemC 使用 coroutine (`wait()`) 模擬時脈週期，比 Verilog 的事件驅動更抽象。

### SystemC `SC_THREAD` 替代方案
現代 SystemC 更傾向使用 `SC_THREAD` 配合顯式 `wait(clk.posedge_event())`：
```cpp
SC_THREAD(seq_proc);
sensitive << clk.pos();

void seq_proc() {
    while (true) {
        if (!rst_n.read()) { q = 0; }
        else { q = d.read(); }
        wait(clk.posedge_event());
    }
}
```

## 4. 鎖存器 (Latch) 與寄存器 (Register)

### Latch (電平敏感)
```verilog
always @(a or b or enable)
    if (enable) q = a + b;
    // else 保持原值 (隱含 latch)
```

**對應**: `SC_METHOD` 配合不完整的 `if-else`:
```cpp
void latch_proc() {
    if (enable.read()) q = a.read() + b.read();
    // 隱含的記憶效應
}
```

### Register (邊緣觸發)
區別關鍵在於 `wait()` 的使用與敏感度設定。

## 5. 三態匯流排 (Tri-State Bus)
```verilog
assign bus = enable ? data : 1'bz;
```

**對應**: SystemC `sc_logic` + `sc_signal_resolved`:
```cpp
sc_logic data_out;
if (enable.read() == SC_LOGIC_1)
    bus.write(data_out);
else
    bus.write(SC_LOGIC_Z);  // 高阻態
```

## 6. 非阻塞賦值差異
| 語言 | 語法 | 語義 |
|------|------|------|
| Verilog | `q <= d` | 非阻塞，立即取樣，週期結束更新 |
| SystemC | `q.write(d)` | 請求更新，Delta 週期結束生效 |
| SystemC | `q = d` | C++ 賦值，立即生效 (僅在 method 內部暫存時使用) |

**關鍵區別**: 
- Verilog 的 `<=` 在 `always` 塊中統一於時間步進 (time step) 結束時更新。
- SystemC 的 `write()` 在 Update Phase (Delta Cycle) 更新。

## 7. 原始碼參考與對照表
- Verilog 轉換工具：Verilator、Synopsys VCS 可將 Verilog 編譯為 SystemC。
- SystemC 綜合工具：Cadence Stratus、Synopsys Synphony 可將 SystemC 綜合為 RTL。

---
*Note: 本文基於 IEEE 1666-2011 SystemC 標準與 Verilog-2001 語義對照。*
