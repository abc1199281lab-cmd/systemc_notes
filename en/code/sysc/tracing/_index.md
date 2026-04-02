# sysc/tracing/ - Waveform Tracing Subsystem

> The signal recording mechanism during SystemC simulation, which exports signal value changes over time to VCD or WIF format files for subsequent analysis with waveform viewing tools.

## Everyday Analogy

Imagine you are watching a football match. **Tracing** is like a "live text commentary" of the game -- recording the position and state of each player (signal) at every moment. After the game, you can use this record to reconstruct what happened at any point in time.

- **sc_trace_file** = The commentator (an abstract role, responsible for "recording")
- **sc_trace_file_base** = The commentator's standard operating procedure (opening files, time calibration, when to record)
- **vcd_trace_file** = A commentator who writes records in "VCD format"
- **wif_trace_file** = A commentator who writes records in "WIF format"
- **sc_trace()** = Telling the commentator "please keep an eye on this player"

## Subsystem Overview

```mermaid
flowchart TD
    subgraph User Interface
        API[sc_trace.h<br/>sc_trace global functions]
    end

    subgraph Infrastructure
        BASE_H[sc_trace_file_base.h<br/>shared tracing base class]
        BASE_C[sc_trace_file_base.cpp<br/>timescale management / lifecycle]
        IDS[sc_tracing_ids.h<br/>error message IDs]
    end

    subgraph Format Implementations
        VCD_H[sc_vcd_trace.h]
        VCD_C[sc_vcd_trace.cpp<br/>VCD format output]
        WIF_H[sc_wif_trace.h]
        WIF_C[sc_wif_trace.cpp<br/>WIF format output]
    end

    API --> BASE_H
    BASE_H --> IDS
    VCD_H --> BASE_H
    WIF_H --> BASE_H
    VCD_C --> VCD_H
    WIF_C --> WIF_H
```

## Class Inheritance Hierarchy

```mermaid
classDiagram
    sc_trace_file <|-- sc_trace_file_base
    sc_trace_file_base <|-- vcd_trace_file
    sc_trace_file_base <|-- wif_trace_file
    sc_stage_callback_if <|.. sc_trace_file_base

    class sc_trace_file {
        <<abstract>>
        +trace(object, name)*
        +write_comment(comment)*
        +set_time_unit(v, tu)*
        +delta_cycles(flag)
        #cycle(delta_cycle)*
        #event_trigger_stamp(event)
    }

    class sc_trace_file_base {
        #fp : FILE*
        #trace_unit_fs : unit_type
        #kernel_unit_fs : unit_type
        +filename()
        +delta_cycles()
        +set_time_unit(v, tu)
        #initialize()
        #open_fp()
        #do_initialize()*
        #add_trace_check(name)
        #timestamp_in_trace_units(high, low)
        -stage_callback(stage)
    }

    class vcd_trace_file {
        +traces : vector~vcd_trace*~
        +obtain_name()
        #do_initialize()
        #cycle(delta_cycle)
        -vcd_name_index : unsigned
    }

    class wif_trace_file {
        +traces : vector~wif_trace*~
        +obtain_name()
        #do_initialize()
        #cycle(delta_cycle)
        -wif_name_index : unsigned
    }
```

## VCD Internal Trace Object Hierarchy

```mermaid
classDiagram
    vcd_trace <|-- vcd_T_trace~T~
    class vcd_trace {
        <<abstract>>
        +name : string
        +vcd_name : string
        +vcd_var_type : vcd_enum
        +bit_width : int
        +write(f)*
        +changed()*
        +print_variable_declaration_line(f, name)
        +print_data_line(f, rawdata)
        +strip_leading_bits(buf)$
    }
    class vcd_T_trace~T~ {
        -object : const T&
        -old_value : T
        +write(f)
        +changed() bool
    }
```

## Tracing Flow

```mermaid
sequenceDiagram
    participant User as User Code
    participant API as sc_trace()
    participant TF as vcd_trace_file
    participant Kernel as sc_simcontext

    Note over User,Kernel: === Setup Phase (before simulation starts) ===
    User->>API: sc_create_vcd_trace_file("dump")
    API->>TF: new vcd_trace_file("dump")
    User->>API: sc_trace(tf, signal, "clk")
    API->>TF: trace(signal, "clk")
    TF->>TF: traces.push_back(new vcd_T_trace)

    Note over User,Kernel: === Simulation Phase ===
    Kernel->>TF: stage_callback(SC_PRE_TIMESTEP)
    TF->>TF: cycle(false)
    TF->>TF: initialize() (first time only)
    loop for each trace
        TF->>TF: changed()?
        TF->>TF: write(fp)
    end

    Note over User,Kernel: === Teardown Phase ===
    User->>API: sc_close_vcd_trace_file(tf)
    API->>TF: delete tf
    TF->>TF: fclose(fp)
```

## File List

| File | Description |
|------|-------------|
| [sc_trace.md](sc_trace.md) | Tracing public API: `sc_trace_file` abstract class and global `sc_trace()` functions |
| [sc_trace_file_base.md](sc_trace_file_base.md) | Tracing file shared base class: timescale, file lifecycle, callback mechanism |
| [sc_tracing_ids.md](sc_tracing_ids.md) | Tracing subsystem error/warning message ID definitions |
| [sc_vcd_trace.md](sc_vcd_trace.md) | VCD (Value Change Dump) format tracing implementation |
| [sc_wif_trace.md](sc_wif_trace.md) | WIF (Waveform Interchange Format) format tracing implementation |

## Related Subsystems

- `sysc/kernel/` -- Simulation kernel, providing `sc_simcontext`, `sc_event`, `sc_time`
- `sysc/communication/` -- Signal interface `sc_signal_in_if`, trace functions can directly trace signals
- `sysc/datatypes/` -- Various data types (`sc_logic`, `sc_bv_base`, etc.), tracing must support all types
- `sysc/utils/` -- Error reporting mechanism `sc_report`
