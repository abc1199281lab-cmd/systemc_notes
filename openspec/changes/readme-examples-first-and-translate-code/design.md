## Context

Follow-up to `restructure-i18n` which established the zh-TW/en directory structure. READMEs currently suggest topdown-first reading, but practitioners prefer examples-first. en/code/ was deferred and now needs translating for full parity.

## Goals / Non-Goals

**Goals:**
- README reading order reflects practitioner workflow: examples → topdown as needed → code deep-dive
- All 184 zh-TW/code/ files translated to en/code/ with identical structure
- Both language directories fully in sync

**Non-Goals:**
- Rewriting or improving the code analysis content itself
- Adding new documentation

## Decisions

### 1. Examples-first reading order
README will suggest: start with examples/ to see SystemC in action, refer to topdown/ when you need conceptual context, dive into code/ for implementation details.

**Rationale**: Most SystemC users are practitioners who learn by running examples, not reading architecture docs.

### 2. Parallel agent translation for code/
Same approach as restructure-i18n: launch parallel agents per subdirectory to translate code/ files.

**Rationale**: 184 files is too many to do sequentially. Subdirectory-level parallelism worked well last time.

## Risks / Trade-offs

- **Translation volume** → 184 files is large but manageable with parallel agents. Same rules apply: keep SystemC terms untranslated, preserve code blocks.
