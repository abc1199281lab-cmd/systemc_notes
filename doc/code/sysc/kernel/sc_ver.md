# SystemC Version Analysis

> **File**: `ref/systemc/src/sysc/kernel/sc_ver.cpp`

## 1. Overview
This file contains the definition of global version strings, copyright messages, and verification checks.

## 2. Version Information
- Defines `systemc_version` string (e.g., "SystemC 2.3.4 ...").
- Defines components: `sc_version_major`, `sc_version_minor`, `sc_version_patch`.
- `sc_copyright()`: Returns the standard Accellera copyright string.

## 3. Link-Time/Run-Time Checks
- **`SC_API_PERFORM_CHECK_`**: A macro used to ensure that the application and the library were compiled with consistent settings (e.g., `SC_DEFAULT_WRITER_POLICY`).
- If mismatches are detected (e.g., app uses one policy, lib uses another), a fatal error is reported to prevent subtle ABI bugs.

## 4. Key Takeaways
1.  **ABI Safety**: The `sc_api_version_...` constructor logic performs crucial sanity checks during program startup (static initialization time) to ensure binary compatibility.
