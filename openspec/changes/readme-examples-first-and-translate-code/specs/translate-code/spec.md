## ADDED Requirements

### Requirement: English translation of code/ directory
The system SHALL provide English translations of all 184 files in zh-TW/code/ at en/code/, maintaining identical directory structure and file names.

#### Scenario: Code files mirror zh-TW
- **WHEN** comparing en/code/ with zh-TW/code/
- **THEN** every .md file in zh-TW/code/ SHALL have a corresponding English .md file in en/code/

#### Scenario: Technical terms preserved
- **WHEN** translating SystemC technical terms (class names, API names, macros)
- **THEN** the English version SHALL keep these terms untranslated

#### Scenario: Code blocks preserved
- **WHEN** translating code analysis docs containing C++ code blocks
- **THEN** code blocks SHALL remain identical; only surrounding explanatory text SHALL be translated

#### Scenario: Mermaid diagrams translated
- **WHEN** a code doc contains mermaid diagram labels in Chinese
- **THEN** the English version SHALL translate the mermaid labels to English
