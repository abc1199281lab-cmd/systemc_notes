# TLM Instance Specific Extensions Analysis

> **File**: `ref/systemc/src/tlm_utils/instance_specific_extensions.h`

## 1. Overview
Instance Specific Extensions (ISPE) allow a module to attach data to a transaction that checks are **private** to that module instance. Standard TLM extensions are global; if Module A attaches an extension, Module B can see it. ISPEs solve this visibility/privacy issue.

## 2. Mechanism
- **The Carrier**: A special extension called `instance_specific_extension_carrier` is attached to the generic payload. This carrier holds a container of private extensions.
- **The Accessor**: Each module instance that wants to use ISPEs must have an `instance_specific_extension_accessor` member.
- **Accessing**:
  ```cpp
  // Set
  m_accessor(trans).set_extension(my_ext);
  
  // Get
  MyExt* ext = m_accessor(trans).get_extension<MyExt>();
  ```

## 3. How It Works
1.  When `m_accessor(trans)` is called, it checks if the transaction already has a carrier.
2.  If not, it creates one and attaches it.
3.  It then returns a helper object (`instance_specific_extensions_per_accessor`) that indexes into the container using the accessor's unique ID.
4.  This ensures that Module A's accessor looks at slot A, and Module B's accessor looks at slot B, even though they attach to the same transaction.

## 4. Usage Scenarios
- **Routers/Mergers**: A router might need to attach routing information (e.g., "this came from port 3") to a transaction as it forwards it. It doesn't want other components to rely on or check this internal implementation detail.
- **Legacy Bridges**: Storing original protocol information when bridging between different bus standards.
