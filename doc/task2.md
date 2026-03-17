# SystemC Examples 文件產生任務

## Context
- 讀者: 純軟體工程師（不懂硬體），用軟體類比解釋硬體 IP
- 每個 IP/範例需要時附 `spec.md` 說明硬體規格（用軟體類比）
- 每告一個段落（例如一個phase), commit once and push to remote
- 好的 diagram (UML, dependencies, flow), 兼容 Obsidian
- top down 加入 learning path
- 繁體中文優先（程式碼/技術關鍵字用英文）

## Naming rule
- Bottom-up: `ref/systemc/examples/<case>/*` → `doc_v2/examples/code/<case>/*.md`
- Spec: `doc_v2/examples/code/<case>/spec.md`
- Top-down: `doc_v2/examples/topdown/{topic}.md`

## 階段進度

| 階段 | 子系統 | 檔案數 | 狀態 |
|------|--------|--------|------|
| Setup | _index.md + 目錄結構 | - | ✅ |
| P1 | sysc/simple_fifo | 1 | ✅ |
| P2 | sysc/rsa, sysc/simple_perf | 2 | ✅ |
| P3 | sysc/pipe | 11 | ✅ |
| P4 | sysc/fir | 14 | ✅ |
| P5 | sysc/fft | 14 | ✅ |
| P6 | sysc/pkt_switch | 13 | ✅ |
| P7 | sysc/simple_bus | 23 | ✅ |
| P8 | sysc/risc_cpu | 22 | ✅ |
| P9 | sysc/2.1 | 8 | ✅ |
| P10 | sysc/2.3, sysc/2.4 | 10 | ✅ |
| P11 | sysc/async_suspend | 5 | ✅ |
| P12 | tlm/common | 40 | ✅ |
| P13 | tlm/lt, tlm/lt_dmi | 10 | ✅ |
| P14 | tlm/lt_temporal_decouple, lt_mixed_endian, lt_extension_mandatory | 21 | ✅ |
| P15 | tlm/at_1_phase, at_2_phase, at_4_phase | 15 | ✅ |
| P16 | tlm/at_extension_optional, at_mixed_targets, at_ooo | 17 | ✅ |
| P17 | Top-down 概念文件 | - | ✅ |
| Final | 整合 & 全域架構圖 | - | ✅ |
