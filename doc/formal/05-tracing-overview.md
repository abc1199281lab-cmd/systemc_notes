# SystemC Tracing and Utils Module Analysis

> Understanding waveform tracing (VCD), reporting, and utility functions
> 
> Date: 2026-02-08

---

## 1. Tracing Overview

SystemC provides signal tracing to generate waveform files for debugging and analysis.

**Supported Formats:**
- **VCD** (Value Change Dump) - Industry standard, supported by all waveform viewers
- **WIF** (Waveform Interchange Format) - Older format

**File**: `ref/systemc/src/sysc/tracing/sc_trace.h`

---

## 2. VCD Tracing Architecture

### Class Hierarchy:

```
sc_trace_file (abstract base)
    └── sc_trace_file_base
            └── vcd_trace_file
            └── wif_trace_file
```

### sc_trace_file: Abstract Interface

```cpp
class SC_API sc_trace_file
{
public:
    // Trace various types (pure virtual)
    virtual void trace( const bool& object, const std::string& name ) = 0;
    virtual void trace( const sc_dt::sc_logic& object, const std::string& name ) = 0;
    virtual void trace( const sc_dt::sc_bv_base& object, const std::string& name ) = 0;
    virtual void trace( const sc_dt::sc_int_base& object, const std::string& name ) = 0;
    virtual void trace( const sc_dt::sc_uint_base& object, const std::string& name ) = 0;
    virtual void trace( const sc_dt::sc_signed& object, const std::string& name ) = 0;
    virtual void trace( const sc_dt::sc_unsigned& object, const std::string& name ) = 0;
    
    // Output comment to trace file
    virtual void write_comment( const std::string& comment ) = 0;
    
    // Enable delta cycle tracing
    virtual void delta_cycles( bool flag );
    
    // Set time unit
    virtual void set_time_unit( double v, sc_time_unit tu ) = 0;

protected:
    // Write trace for this cycle
    virtual void cycle( bool delta_cycle ) = 0;
};
```

### Creating and Using Trace Files:

```cpp
#include <systemc.h>

int sc_main(int argc, char* argv[])
{
    // Create trace file
    sc_trace_file* tf = sc_create_vcd_trace_file("my_waveform");
    
    // Set time unit (optional)
    tf->set_time_unit(1, SC_NS);
    
    // Create signals
    sc_signal<bool> clk("clk");
    sc_signal<sc_uint<8>> data("data");
    
    // Trace signals
    sc_trace(tf, clk, "clock");
    sc_trace(tf, data, "data_bus");
    
    // Run simulation
    sc_start(100, SC_NS);
    
    // Close trace file
    sc_close_vcd_trace_file(tf);
    
    return 0;
}
```

---

## 3. vcd_trace_file Implementation

**File**: `ref/systemc/src/sysc/tracing/sc_vcd_trace.h`

```cpp
class vcd_trace_file : public sc_trace_file_base
{
public:
    vcd_trace_file(const char *name);
    ~vcd_trace_file();

protected:
    // Trace implementations for each type
    virtual void trace(const bool& object, const std::string& name);
    virtual void trace(const sc_dt::sc_logic& object, const std::string& name);
    virtual void trace(const unsigned char& object, const std::string& name, int width);
    virtual void trace(const unsigned int& object, const std::string& name, int width);
    virtual void trace(const sc_dt::sc_bv_base& object, const std::string& name);
    virtual void trace(const sc_dt::sc_lv_base& object, const std::string& name);
    // ... more types

private:
    unsigned vcd_name_index;           // Variable counter for naming
    unit_type previous_time_units_low; // Last timestamp low
    unit_type previous_time_units_high;// Last timestamp high
    
    std::vector<vcd_trace*> traces;    // All traced variables
    std::string obtain_name();         // Generate VCD symbol name
    
    void do_initialize();              // Write VCD header
    void print_time_stamp(unit_type now_units_high, unit_type now_units_low) const;
    void cycle(bool delta_cycle);      // Called each cycle
};
```

### VCD File Format:

VCD files contain:

1. **Header**: Date, version, timescale
2. **Variable Definitions**: Signal names and identifiers
3. **Value Changes**: Timestamped value updates

Example VCD output:
```vcd
$date
   Sun Feb 08 06:40:00 2026
$end
$version
   SystemC 2.3.3
$end
$timescale
   1ns
$end

$scope module SystemC $end
$var wire 1 ! clock $end
$var wire 8 " data_bus $end
$upscope $end
$enddefinitions $end

#0
0!
b00000000 "

#5
1!

#10
0!
b10101010 "
```

### RTL Knowledge: Why VCD?

**VCD is the universal format for digital waveform viewing:**

| Tool | VCD Support |
|:-----|:------------|
| GTKWave | Native |
| ModelSim | Import/Export |
| Verilator | Export only |
| Vivado | Import |
| Commercial simulators | All support |

**Uses:**
- Debug RTL vs reference model mismatches
- Analyze timing relationships
- Verify protocol compliance
- Create test case documentation

---

## 4. Trace Implementation Details

### vcd_T_trace Template:

```cpp
template<class T>
class vcd_T_trace : public vcd_trace
{
    const T& object;  // Reference to traced object

public:
    vcd_T_trace(const T& object_, const std::string& name_, 
                const std::string& vcd_name_, vcd_enum type_)
        : object(object_), vcd_trace(name_, vcd_name_, type_)
    {}

    void write(FILE* f)  // Write current value to file
    {
        // Format depends on type (bool, logic, vector)
        if (object == previous_value) return;  // Only write changes
        // ... write value
        previous_value = object;
    }
};
```

### Delta Cycle Tracing:

```cpp
virtual void delta_cycles( bool flag )
{
    trace_delta_cycles = flag;
}
```

When enabled, delta cycles appear as zero-time transitions in the VCD:

```vcd
#100           // Time 100ns
1clock         // Clock rises

#100           // Same time (delta cycle)
b11110000 data // Data updates after delta update
```

This helps debug race conditions and delta-cycle behavior.

---

## 5. Utils: Report Handling

**Files**: `ref/systemc/src/sysc/utils/`

### Report Severity Levels:

```cpp
enum sc_severity {
    SC_INFO = 0,    // Informational message
    SC_WARNING,     // Warning, continues simulation
    SC_ERROR,       // Error, may continue or abort
    SC_FATAL        // Fatal error, aborts simulation
};
```

### Report Actions:

```cpp
enum sc_actions {
    SC_UNSPECIFIED = 0x0000,  // Use default
    SC_DO_NOTHING  = 0x0001,  // Ignore
    SC_THROW       = 0x0002,  // Throw exception
    SC_LOG         = 0x0004,  // Log to file
    SC_DISPLAY     = 0x0008,  // Print to stdout
    SC_STOP        = 0x0010,  // Call sc_stop()
    SC_INTERRUPT   = 0x0020,  // Interrupt (not implemented)
    SC_ABORT       = 0x0040   // Abort simulation
};
```

### sc_report_handler:

```cpp
class SC_API sc_report_handler
{
public:
    // Register message types with actions
    static void set_actions(const char* msg_type, 
                           sc_actions = SC_UNSPECIFIED);
    
    // Set catch and report actions
    static void catch_actions(sc_actions);
    static void throw_actions(sc_actions);
    
    // Suppress specific message IDs
    static void suppress(sc_msg_type);
    static void suppress(sc_msg_type, int msg_id);
    
    // Force actions for specific message
    static void force(sc_msg_type, sc_actions);
    static void force(sc_msg_type, int msg_id, sc_actions);
};
```

### Usage Example:

```cpp
// Suppress all INFO messages
sc_report_handler::set_actions(SC_INFO, SC_DO_NOTHING);

// Make all WARNINGs stop simulation
sc_report_handler::set_actions(SC_WARNING, SC_STOP);

// Report with message ID
SC_REPORT_INFO("/MY_MODULE/INIT", "Starting initialization");
SC_REPORT_WARNING("/MY_MODULE/TIMING", "Setup time violated");
```

---

## 6. Common Message IDs

| Message ID | Severity | Description |
|:-----------|:---------|:------------|
| SC_ID_NO_DEFAULT_EVENT_ | Warning | Port has no default event |
| SC_ID_NOT_REGISTERING_ID_ | Warning | Failed to register ID |
| SC_ID_WRONG_PROCESS_ | Error | Wrong process type |
| SC_ID_SC_MODULE_NAME_RESERVED_ | Error | Module name reserved |
| SC_ID_INSERT_PORT_ | Error | Cannot insert port after elaboration |
| SC_ID_BIND_IF_TO_PORT_ | Error | Interface already bound |
| SC_ID_COMPLETE_BINDING_ | Error | Port binding incomplete |

---

## 7. Utilities: sc_stop and sc_pause

```cpp
// Stop simulation after current delta cycle
void sc_stop();

// Pause simulation (implementation defined)
void sc_pause();

// Check if simulation is running
bool sc_is_running();

// Check if simulation is paused
bool sc_is_paused();

// Get current simulation status
sc_status sc_get_status();
```

---

## 8. Summary: Tracing Best Practices

| Practice | Benefit |
|:---------|:--------|
| Always trace clock | Essential for timing analysis |
| Trace state machines | Easy protocol verification |
| Use meaningful names | Self-documenting waveforms |
| Enable delta cycles | Debug subtle timing issues |
| Group related signals | Use $scope in VCD |
| Limit traced signals | Reduce file size, improve performance |
| Add comments | Document test conditions |

---

## 9. RTL Knowledge: Waveform Debugging

**Typical Waveform Debug Flow:**

1. **Run simulation** with tracing enabled
2. **Open VCD** in GTKWave or similar tool
3. **Add signals** to display window
4. **Zoom to time** of interest
5. **Compare** RTL vs SystemC reference model
6. **Identify** mismatch point
7. **Fix** RTL or model

**Common Issues Found:**

| Issue | Waveform Symptom |
|:------|:-----------------|
| Missing reset | X values propagate |
| Clock skew | Setup/hold violations |
| Protocol error | Data arrives at wrong phase |
| Race condition | Delta cycle ordering issue |
| X propagation | Uninitialized outputs |
