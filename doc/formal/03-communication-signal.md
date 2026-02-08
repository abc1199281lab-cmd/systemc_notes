# SystemC Communication Module Analysis

> Deep dive into SystemC communication mechanisms: Interface, Port, Signal, Channel, Clock, Mutex, Semaphore
> 
> Date: 2026-02-08

---

## 1. Overview: SystemC Communication Architecture

SystemC communication follows a layered architecture:

1. **sc_interface** - Abstract interface definition
2. **sc_port** - Connection points in modules
3. **sc_prim_channel** - Basic channel implementation
4. **Concrete channels** - sc_signal, sc_fifo, sc_mutex, etc.

### RTL Mapping

| SystemC | Verilog | Description |
|:--------|:--------|:------------|
| sc_signal | wire/reg | Hardware signals |
| sc_fifo | FIFO buffer | Hardware FIFO |
| sc_mutex | Arbiter | Hardware arbitration |
| sc_clock | Clock generator | System clock |

---

## 2. sc_interface: Abstract Base Class

**File**: `ref/systemc/src/sysc/communication/sc_interface.h`

```cpp
class SC_API sc_interface
{
public:
    virtual void register_port( sc_port_base& port_, const char* if_typename_ );
    virtual const sc_event& default_event() const;
    virtual ~sc_interface();
protected:
    sc_interface();
};
```

### Key Design Concepts:

- **Pure Abstract Class**: Defines what an interface should provide
- **register_port**: Allows interface to track connected ports
- **default_event()**: Provides standard change event for sensitivity
- **Virtual Inheritance**: Must use virtual when directly inheriting

### Why This Design?

Think of sc_interface like USB specification - it defines what functions should exist (like read/write) but doesn't specify implementation. Different channels (sc_signal, sc_fifo) implement these interfaces differently.

---

## 3. sc_port: Port Implementation

**File**: `ref/systemc/src/sysc/communication/sc_port.h`

### Class Hierarchy:

```
sc_port_base (non-template base)
    └── sc_port_b<IF> (template with interface type)
            └── sc_port<IF, N, POL> (user-facing class)
```

### Port Binding Policies:

```cpp
enum sc_port_policy
{
    SC_ONE_OR_MORE_BOUND,   // Default: at least one interface
    SC_ZERO_OR_MORE_BOUND,  // Optional: can be unbound
    SC_ALL_BOUND            // All positions must be bound
};
```

### Key Mechanisms:

| Mechanism | Description |
|:----------|:------------|
| operator-> | Direct access to interface methods |
| operator[] | Access multi-bound interfaces |
| sc_bind_info | Deferred binding information |
| complete_binding() | Validation at elaboration end |

### sc_bind_info Structure:

```cpp
struct sc_bind_info {
    int                 m_max_size;       // Maximum bindings
    sc_port_policy      m_policy;         // Binding policy
    std::vector<sc_bind_elem*> vec;      // Binding elements
    ef_vector           thread_vec;       // Sensitivity for threads
    ef_vector           method_vec;       // Sensitivity for methods
};
```

Bindings are recorded during construction but validated during elaboration phase.

---

## 4. sc_signal: Hardware Signal Implementation

**File**: `ref/systemc/src/sysc/communication/sc_signal.h`

### Core Implementation:

```cpp
template< class T, sc_writer_policy POL >
class sc_signal_t : public sc_signal_inout_if<T>
{
protected:
    T m_cur_val;    // Current value
    T m_new_val;    // Next value (waiting for update)

public:
    const T& read() const { return m_cur_val; }
    
    void write( const T& value_ ) {
        m_new_val = value_;
        if( value_changed || policy_type::needs_update() ) {
            request_update();  // Schedule update phase execution
        }
    }
    
    void update() {
        if( !( m_new_val == m_cur_val ) ) {
            m_cur_val = m_new_val;
            // Trigger value_changed_event
        }
    }
};
```

### Request-Update Pattern:

The pattern ensures deterministic signal updates:

1. **Evaluate Phase**: Multiple processes may write to signal, only m_new_val changes
2. **Update Phase**: m_cur_val is updated once, event triggered if value changed
3. **Next Delta**: Processes waiting on signal events wake up

### Writer Policies:

```cpp
enum sc_writer_policy {
    SC_ONE_WRITER,          // Only one writer (default)
    SC_MANY_WRITERS,        // Multiple writers, check conflicts
    SC_UNCHECKED_WRITERS    // No checking
};
```

### RTL Mapping:

| Policy | Hardware Equivalent |
|:---------|:--------------------|
| SC_ONE_WRITER | Single-driver reg |
| SC_MANY_WRITERS | Tri-state bus, resolved signals |

### sc_signal<bool> Specialization:

```cpp
template< sc_writer_policy POL >
class sc_signal<bool,POL> : public sc_signal_t<bool,POL>
{
    const sc_event& posedge_event() const;  // Rising edge
    const sc_event& negedge_event() const;  // Falling edge
    
    bool posedge() const { return event() && m_cur_val; }
    bool negedge() const { return event() && !m_cur_val; }
};
```

Provides edge detection - essential for clock and reset signals.

---

## 5. sc_clock: Clock Generator

**File**: `ref/systemc/src/sysc/communication/sc_clock.h`

```cpp
class SC_API sc_clock : public sc_signal<bool,SC_ONE_WRITER>
{
    sc_time  m_period;        // Clock period
    double   m_duty_cycle;    // Duty cycle ratio
    sc_time  m_start_time;    // First edge time
    bool     m_posedge_first; // True for rising edge first
    
    sc_event m_next_posedge_event;
    sc_event m_next_negedge_event;
    
    void posedge_action();    // Rising edge handler
    void negedge_action();    // Falling edge handler
};
```

### Clock Generation:

```cpp
inline void sc_clock::posedge_action()
{
    m_next_negedge_event.notify_internal( m_negedge_time );
    m_new_val = true;
    request_update();
}
```

Each edge schedules the opposite edge, creating continuous clock waveform.

---

## 6. sc_fifo: First-In-First-Out Queue

**File**: `ref/systemc/src/sysc/communication/sc_fifo.h`

```cpp
template <class T>
class sc_fifo : public sc_fifo_in_if<T>,
                public sc_fifo_out_if<T>,
                public sc_prim_channel
{
    int m_size;     // Buffer size
    T*  m_buf;      // Circular buffer
    int m_free;     // Available slots
    int m_ri;       // Read index
    int m_wi;       // Write index
    
    sc_event m_data_read_event;
    sc_event m_data_written_event;

public:
    void read( T& );        // Blocking read
    bool nb_read( T& );     // Non-blocking read
    void write( const T& ); // Blocking write
    bool nb_write( const T& ); // Non-blocking write
    
    int num_available() const;
    int num_free() const;
};
```

### Update Mechanism:

```cpp
void sc_fifo<T>::update()
{
    if( m_num_read > 0 )
        m_data_read_event.notify(SC_ZERO_TIME);
    if( m_num_written > 0 )
        m_data_written_event.notify(SC_ZERO_TIME);
    
    m_num_readable = m_size - m_free;
    m_num_read = 0;
    m_num_written = 0;
}
```

### Hardware FIFO Correspondence:

| Software (sc_fifo) | Hardware FIFO |
|:-------------------|:--------------|
| num_free() == 0 | full signal |
| num_available() == 0 | empty signal |
| write() blocking | Backpressure |
| read() blocking | Flow control |

---

## 7. sc_mutex: Mutual Exclusion Lock

**File**: `ref/systemc/src/sysc/communication/sc_mutex.h`

```cpp
class SC_API sc_mutex : public sc_mutex_if, public sc_object
{
    sc_process_b* m_owner;    // Current owner
    sc_event      m_free;     // Event when released

public:
    int lock();      // Block until acquired
    int trylock();   // Return -1 if cannot lock
    int unlock();    // Release, return -1 if not owner
};
```

### Implementation:

```cpp
int sc_mutex::lock()
{
    while( in_use() ) {
        sc_core::wait( m_free );  // Wait for release
    }
    m_owner = sc_get_current_process_b();
    return 0;
}
```

### RTL Mapping:

sc_mutex corresponds to hardware arbiters that grant exclusive access to shared resources like:
- Memory controllers
- Bus masters
- Shared peripherals

---

## 8. sc_prim_channel: Primitive Channel Base

**File**: `ref/systemc/src/sysc/communication/sc_prim_channel.h`

```cpp
class SC_API sc_prim_channel : public sc_object
{
    friend class sc_prim_channel_registry;
    
    sc_prim_channel_registry* m_registry;
    sc_prim_channel*          m_update_next_p;  // Update list pointer

public:
    inline void request_update();      // Request update phase call
    void async_request_update();       // External update request
    
    virtual void update();             // Override for update behavior
    
protected:
    // Convenience wait methods for threads
    void wait( const sc_event& e ) { sc_core::wait(e, simcontext()); }
    void wait( const sc_time& t ) { sc_core::wait(t, simcontext()); }
};
```

### Update List Management:

```cpp
class sc_prim_channel_registry
{
    sc_prim_channel* m_update_list_p;      // Head of update list
    sc_prim_channel* m_update_list_end;    // Terminator
    
    inline void request_update( sc_prim_channel& prim_channel_ )
    {
        prim_channel_.m_update_next_p = m_update_list_p;
        m_update_list_p = &prim_channel_;
    }
    
    void perform_update();  // Called in update phase
};
```

---

## 9. Summary: Communication Mechanism Comparison

| Mechanism | Use Case | Synchronization | RTL Equivalent |
|:----------|:---------|:----------------|:-------------|
| sc_signal | Data flow between processes | Delta cycle update | wire/reg |
| sc_fifo | Buffered data transfer | Blocking/non-blocking | Hardware FIFO |
| sc_mutex | Exclusive resource access | Lock/unlock | Arbiter |
| sc_semaphore | Limited resource pool | Count-based | Token bucket |
| sc_clock | Periodic timing | Timed events | Clock generator |

---

## 10. RTL Knowledge: Why Request-Update Pattern?

In hardware simulation, this corresponds to **non-blocking assignments** in Verilog:

```verilog
// Verilog non-blocking assignment
always @(posedge clk)
    q <= d;  // All reads use old value, updates at end of time step
```

SystemC sc_signal::write() is equivalent to <= (non-blocking)
SystemC sc_signal::read() in same delta returns old value

This ensures deterministic execution order regardless of process scheduling.
