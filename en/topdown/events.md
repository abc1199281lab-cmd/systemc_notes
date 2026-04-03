# Event Mechanism

## Everyday Analogy: Phone Notifications

SystemC's event mechanism is like a phone's notification system:

- **Event (sc_event)** = Your phone's notification sound — "Ding! Something happened"
- **Waiting for an event (wait)** = You're waiting for a notification from an app — "Let me know when my package arrives"
- **Triggering an event (notify)** = Someone presses the send button — "Package delivered, sending notification"
- **Sensitivity list (sensitivity)** = The apps you've enabled notifications for in your settings

You don't need to check every second whether the package has arrived (polling).
You just wait for the notification to ring (event-driven). That's the essence of the event mechanism.

---

## What Is an Event?

`sc_event` is the most fundamental synchronization primitive in SystemC. It **carries no data** —
it's simply a "flare" that tells the simulation engine: "Something happened, the relevant processes should wake up."

```mermaid
sequenceDiagram
    participant ProcessA as Process A<br/>(Sender)
    participant Event as sc_event
    participant ProcessB as Process B<br/>(Waiter)

    ProcessB->>Event: wait(event) - I'm waiting
    Note over ProcessB: Enters sleep...

    ProcessA->>Event: event.notify() - Something happened!
    Event->>ProcessB: Wake up!
    Note over ProcessB: Resumes execution
```

---

## Three Notification Modes

Events can trigger notifications with three different "delays":

```mermaid
graph TD
    N[event.notify] --> IMM["Immediate notification<br/>notify()"]
    N --> DELTA["Delta notification<br/>notify(SC_ZERO_TIME)"]
    N --> TIMED["Timed notification<br/>notify(10, SC_NS)"]

    IMM --> IMM_DESC["Wakes waiting processes<br/>immediately in the current<br/>evaluate phase"]
    DELTA --> DELTA_DESC["Wakes processes in the<br/>evaluate phase of the<br/>next delta cycle"]
    TIMED --> TIMED_DESC["Wakes processes at<br/>a specified future time"]

    style IMM fill:#ffcdd2
    style DELTA fill:#fff3e0
    style TIMED fill:#e3f2fd
```

### Immediate Notification `notify()`

```cpp
event.notify();  // Right now! Immediately!
```

Analogy: You tap your colleague on the shoulder in the office and say, "Hey, this is done."
Your colleague knows **immediately**.

**Note**: Immediate notification only takes effect during the current evaluate phase.
If no process is waiting, this notification is lost.

### Delta Notification `notify(SC_ZERO_TIME)`

```cpp
event.notify(SC_ZERO_TIME);  // Next delta cycle
```

Analogy: You send an internal instant message at work. The other person will see it very soon,
but not in the exact same "instant" you typed it — there's a tiny delay in between.

**This is the most commonly used notification mode**, because it guarantees correct event ordering.

### Timed Notification `notify(time)`

```cpp
event.notify(10, SC_NS);  // After 10 nanoseconds
```

Analogy: You set an alarm to go off in 10 seconds.

---

## Notification Timing Diagram

```mermaid
sequenceDiagram
    participant T as Timeline
    participant A as Process A
    participant E1 as Event1
    participant E2 as Event2
    participant E3 as Event3
    participant B as Process B
    participant C as Process C

    Note over T: t=5ns, delta#0
    A->>E1: notify() [immediate]
    A->>E2: notify(SC_ZERO_TIME) [delta]
    A->>E3: notify(10, SC_NS) [timed]

    E1->>B: Immediately wakes B
    Note over B: B executes in the same delta#0

    Note over T: t=5ns, delta#1
    E2->>C: Wakes C in the next delta
    Note over C: C executes in delta#1

    Note over T: t=15ns, delta#0
    E3->>B: Wakes B at t=15ns
    Note over B: B executes again
```

---

## Static Sensitivity vs Dynamic Sensitivity

### Static Sensitivity

Set during the construction phase; does not change during simulation.

```cpp
SC_METHOD(my_method);
sensitive << clk.pos() << reset;  // Always sensitive to these events
```

Analogy: You turn on notifications for "Gmail" and "Line" in your phone settings —
whenever there's a new email or message, you'll always be notified.

```mermaid
flowchart LR
    CLK[Clock rising edge event] -->|Always triggers| M[my_method]
    RST[Reset event] -->|Always triggers| M

    style M fill:#e3f2fd
    style CLK fill:#c8e6c9
    style RST fill:#c8e6c9
```

### Dynamic Sensitivity

Dynamically decides which events to wait for while a process is running.

```cpp
void my_thread() {
    while (true) {
        wait(event_a);        // This time, wait for event_a
        // ... do something ...
        wait(event_b);        // Next, wait for event_b
        // ... do something else ...
        wait(10, SC_NS);      // Then wait 10ns
    }
}
```

Analogy: While waiting for a package delivery, once it arrives you switch to waiting for
a food delivery notification, then a taxi notification — each time you're waiting for something different.

```mermaid
stateDiagram-v2
    [*] --> WaitEventA: wait(event_a)
    WaitEventA --> HandleA: event_a triggered
    HandleA --> WaitEventB: wait(event_b)
    WaitEventB --> HandleB: event_b triggered
    HandleB --> Wait10ns: wait(10, SC_NS)
    Wait10ns --> WaitEventA: After 10ns
```

### Differences Between SC_METHOD and SC_THREAD

| Property | SC_METHOD | SC_THREAD |
|----------|-----------|-----------|
| Sensitivity | Primarily static | Primarily dynamic |
| Can call `wait()`? | No! | Yes |
| Execution model | Runs to completion each time | Can suspend and resume |
| Analogy | Do one thing when the alarm rings | A long-term worker who can take breaks mid-task |

---

## Event Lists (AND / OR)

Sometimes you want to wait on a combination of multiple events:

### OR List — Wake up when any event fires

```cpp
wait(event_a | event_b | event_c);
```

Analogy: "Package delivery, food delivery, or a friend's call — notify me when any one of them arrives."

### AND List — Wake up only when all events have fired

```cpp
wait(event_a & event_b & event_c);
```

Analogy: "Package delivered **and** food delivered **and** friend called — notify me only when all three are done."

```mermaid
flowchart TD
    subgraph "OR List (event_a | event_b)"
        OA[event_a fired?] -->|Yes| OWAKE[Process wakes up!]
        OB[event_b fired?] -->|Yes| OWAKE
    end

    subgraph "AND List (event_a & event_b)"
        AA[event_a fired?] --> ACHECK{Both<br/>fired?}
        AB[event_b fired?] --> ACHECK
        ACHECK -->|Yes| AWAKE[Process wakes up!]
        ACHECK -->|No| AWAIT[Keep waiting]
    end

    style OWAKE fill:#c8e6c9
    style AWAKE fill:#c8e6c9
    style AWAIT fill:#ffcdd2
```

---

## How Events Drive Simulation

Putting all the concepts together, let's see the role events play in the entire simulation:

```mermaid
flowchart TD
    subgraph "Simulation Engine"
        R[Runnable Queue<br/>Ready processes]
        E[Evaluate<br/>Execute processes]
        U[Update<br/>Update channels]
        T[Timed Event Queue<br/>Future event list]
    end

    subgraph "Process Behavior"
        P1[Write to signal]
        P2[notify event]
        P3[wait event]
    end

    R --> E
    E --> P1
    E --> P2
    E --> P3

    P1 -->|request_update| U
    U -->|Value changed, delta notification| R

    P2 -->|Immediate notification| R
    P2 -->|Delta notification| R
    P2 -->|Timed notification| T

    T -->|Time reached| R

    P3 -->|Suspend process| R

    style R fill:#e3f2fd
    style E fill:#fff3e0
    style U fill:#e8f5e9
    style T fill:#fce4ec
```

---

## Internal Structure of sc_event

```mermaid
classDiagram
    class sc_event {
        -m_name : string
        -m_notify_type : notification_type
        -m_delta_event_index : int
        -m_timed_event_index : int
        -m_methods_static : vector
        -m_methods_dynamic : vector
        -m_threads_static : vector
        -m_threads_dynamic : vector
        +notify()
        +notify(t : sc_time)
        +notify_delayed()
        +notify_delayed(t : sc_time)
        +cancel()
    }

    class sc_event_list {
        -m_events : vector~sc_event~
        -m_and_list : bool
        +size() int
        +auto_delete() bool
    }

    class sc_event_and_list {
    }
    class sc_event_or_list {
    }

    sc_event_list <|-- sc_event_and_list
    sc_event_list <|-- sc_event_or_list
    sc_event_list o-- sc_event : contains
```

---

## Related Modules

| Concept | File | Relationship |
|---------|------|--------------|
| Simulation Engine | [simulation-engine.md](simulation-engine.md) | Events drive the entire simulation engine |
| Scheduling | [scheduling.md](scheduling.md) | The scheduler decides when to process which events |
| Communication | [communication.md](communication.md) | Signals use events internally to notify value changes |
| Module Hierarchy | [hierarchy.md](hierarchy.md) | Processes are defined in modules; processes use events |

### Corresponding Source Code Files

| Source Concept | Code File |
|---------------|-----------|
| sc_event | [doc_v2/code/sysc/kernel/sc_event.md](../code/sysc/kernel/sc_event.md) |
| sc_sensitive | [doc_v2/code/sysc/kernel/sc_sensitive.md](../code/sysc/kernel/sc_sensitive.md) |
| sc_wait | [doc_v2/code/sysc/kernel/sc_wait.md](../code/sysc/kernel/sc_wait.md) |
| sc_method_process | [doc_v2/code/sysc/kernel/sc_method_process.md](../code/sysc/kernel/sc_method_process.md) |
| sc_thread_process | [doc_v2/code/sysc/kernel/sc_thread_process.md](../code/sysc/kernel/sc_thread_process.md) |
| sc_event_finder | [doc_v2/code/sysc/communication/sc_event_finder.md](../code/sysc/communication/sc_event_finder.md) |

---

## Learning Tips

1. **Events carry no data** — it's just a "ding!" notification; data must be passed through signals or other means
2. **Immediate notifications are error-prone** — beginners should prefer delta notifications (`notify(SC_ZERO_TIME)`)
3. **SC_METHOD cannot call `wait()`** — this is one of the most common beginner mistakes
4. **AND list events don't need to fire at the same instant** — they just all need to have fired at some point
5. **Events are one-shot** — once fired, they're gone, unlike signals which retain their value. If you need to "remember" state, use a signal
6. **`cancel()` can only cancel timed notifications** — immediate and delta notifications that have already been issued cannot be cancelled
