## ADDED Requirements

### Requirement: English README
The system SHALL have a `README.md` at the project root written in English, containing project introduction, document navigation, and suggested reading order.

#### Scenario: README.md exists and links to zh-TW version
- **WHEN** a user opens `README.md`
- **THEN** it SHALL contain a language switcher at the top linking to `README_zh-TW.md`

#### Scenario: README.md provides document navigation
- **WHEN** a user reads `README.md`
- **THEN** it SHALL list available documentation directories (`en/`, `zh-TW/`) with descriptions of their contents

### Requirement: Traditional Chinese README
The system SHALL have a `README_zh-TW.md` at the project root written in Traditional Chinese, with equivalent content to `README.md`.

#### Scenario: README_zh-TW.md exists and links to English version
- **WHEN** a user opens `README_zh-TW.md`
- **THEN** it SHALL contain a language switcher at the top linking to `README.md`

#### Scenario: README_zh-TW.md provides document navigation
- **WHEN** a user reads `README_zh-TW.md`
- **THEN** it SHALL list available documentation directories (`zh-TW/`, `en/`) with descriptions of their contents
