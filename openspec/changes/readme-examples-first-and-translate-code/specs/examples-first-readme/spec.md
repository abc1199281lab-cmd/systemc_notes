## ADDED Requirements

### Requirement: README suggests examples-first reading order
Both README.md and README_zh-TW.md SHALL recommend starting with examples/, then referring to topdown/ as needed, then code/ for deep dives.

#### Scenario: English README reading order
- **WHEN** a user reads README.md's "Suggested Reading Order" section
- **THEN** the first step SHALL direct them to examples/ as the starting point
- **THEN** topdown/ SHALL be suggested as optional conceptual reference

#### Scenario: Chinese README reading order
- **WHEN** a user reads README_zh-TW.md's "建議閱讀順序" section
- **THEN** the first step SHALL direct them to examples/ as the starting point
- **THEN** topdown/ SHALL be suggested as optional conceptual reference

### Requirement: Documentation table reflects full en/ coverage
Both READMEs SHALL update the documentation table to show en/ includes code/ (no longer zh-TW only).

#### Scenario: en/ description updated
- **WHEN** a user reads the documentation table
- **THEN** en/ description SHALL include code analysis, not just "top-down concepts, examples, and diagrams"
