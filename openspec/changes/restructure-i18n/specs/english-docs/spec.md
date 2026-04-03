## ADDED Requirements

### Requirement: English translation of topdown docs
The system SHALL provide English translations of all files in `zh-TW/topdown/` at `en/topdown/`, maintaining identical directory structure and file names.

#### Scenario: Topdown files mirror zh-TW
- **WHEN** comparing `en/topdown/` with `zh-TW/topdown/`
- **THEN** every `.md` file in `zh-TW/topdown/` SHALL have a corresponding English `.md` file in `en/topdown/`

#### Scenario: Technical terms preserved
- **WHEN** translating SystemC technical terms (e.g., sc_module, sc_signal, TLM, sc_event)
- **THEN** the English version SHALL keep these terms untranslated, same as the Chinese version

### Requirement: English translation of examples docs
The system SHALL provide English translations of all files in `zh-TW/examples/` at `en/examples/`, maintaining identical directory structure and file names.

#### Scenario: Examples files mirror zh-TW
- **WHEN** comparing `en/examples/` with `zh-TW/examples/`
- **THEN** every `.md` file in `zh-TW/examples/` SHALL have a corresponding English `.md` file in `en/examples/`

#### Scenario: Code blocks preserved
- **WHEN** translating example docs containing C++ code blocks
- **THEN** code blocks SHALL remain identical; only surrounding explanatory text SHALL be translated

### Requirement: English translation of diagrams docs
The system SHALL provide English translations of all files in `zh-TW/diagrams/` at `en/diagrams/`, maintaining identical directory structure and file names.

#### Scenario: Diagrams files mirror zh-TW
- **WHEN** comparing `en/diagrams/` with `zh-TW/diagrams/`
- **THEN** every `.md` file in `zh-TW/diagrams/` SHALL have a corresponding English `.md` file in `en/diagrams/`

#### Scenario: Mermaid diagrams translated
- **WHEN** a diagram file contains mermaid diagram labels in Chinese
- **THEN** the English version SHALL translate the mermaid labels to English
