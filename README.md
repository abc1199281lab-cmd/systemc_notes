# SystemC Learning Notes

[English] | [繁體中文](README_zh-TW.md)

> **Structured learning notes for the SystemC library** — from top-down conceptual architecture to bottom-up source code analysis.

## Target Audience

Software engineers without hardware RTL background who want to deeply understand SystemC. All explanations aim to be simple enough for a high school student to follow.

## Documentation

| Directory | Language | Description |
|-----------|----------|-------------|
| [en/](en/) | English | Concepts, code analysis, examples, and diagrams |
| [zh-TW/](zh-TW/) | 繁體中文 | Concepts, code analysis, examples, and diagrams |

### Content Structure

```
examples/    — SystemC & TLM example walkthroughs (start here!)
topdown/     — Conceptual architecture docs (simulation engine, events, scheduling, etc.)
code/        — Per-file source code analysis
diagrams/    — Architecture overview diagrams
```

## Suggested Reading Order

1. **Start with examples** — browse [examples/](en/examples/) to see SystemC in action. Most developers learn best by studying working code
2. **Refer to concepts as needed** — check [topdown/](en/topdown/) when you want to understand the "why" behind a pattern
3. **Deep dive into source code** — explore [code/](en/code/) for per-file analysis of the SystemC library internals
4. Mermaid diagrams can be viewed on GitHub, VSCode, or any Mermaid-compatible tool

## About

These notes were created to systematically study the [Accellera SystemC](https://systemc.org/) open-source library, covering both the core SystemC kernel and TLM (Transaction-Level Modeling).
