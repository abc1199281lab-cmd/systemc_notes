## Why

Most SystemC developers start by using the library and learning from examples, not by reading conceptual architecture docs. The current README suggests starting with `topdown/learning-path.md`, which doesn't match how practitioners actually learn. Additionally, `en/code/` is missing — the previous change only translated topdown/, examples/, and diagrams/, leaving 184 code analysis files untranslated.

## What Changes

- Update `README.md` and `README_zh-TW.md` suggested reading order: examples first, topdown concepts as needed
- Update documentation table to reflect en/ now includes code/
- Translate all 184 files in `zh-TW/code/` → `en/code/` (sysc + tlm_core + tlm_utils)

## Capabilities

### New Capabilities
- `examples-first-readme`: Update both READMEs to recommend examples as the starting point
- `translate-code`: Translate zh-TW/code/ (184 files) to en/code/ for full bilingual parity

### Modified Capabilities

（無既有 spec 需要修改）

## Impact

- README.md and README_zh-TW.md content changes (reading order, documentation table)
- 184 new files created under en/code/
- AGENTS.md may need minor update to reflect en/ now includes code/
