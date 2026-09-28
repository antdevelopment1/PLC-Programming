# Pressure Receiver Pump Control & Indicator Logic

A PLC programming exercise built in RSLogix Micro Starter Lite to practice digital I/O mapping, pressure-based pump control, truth/state-table development, hysteresis, state persistence, seal-in logic, input qualification, event generation, and ladder-logic refactoring.

---

## 🔍 Overview

This project controls pressure inside a receiver using two pressure switches and a single pump.

The process uses:

- A low-pressure switch that closes at 90 PSI and above
- A high-pressure switch that closes at 110 PSI and above
- A pressure pump used to increase receiver pressure
- A pressure indicator light that illuminates once the low-pressure threshold is reached

The pump must:

- Start when pressure is below 90 PSI
- Continue running after the low-pressure switch closes
- Stop when pressure reaches 110 PSI
- Remain stopped while pressure falls back below 110 PSI
- Restart only after pressure falls below 90 PSI

This creates a hysteresis control pattern using separate start and stop thresholds rather than cycling the pump around a single pressure point.

---

## ⚙️ Platform & Tools

- **PLC:** Allen-Bradley MicroLogix 1100 Series B
- **Programming Software:** RSLogix Micro Starter Lite / RSLogix 500
- **Simulation:** RSLogix Emulate
- **Language:** Ladder Logic

---

## 🗺️ System I/O & Tags

### Physical Inputs

| Address | Tag | Description |
|---|---|---|
| I:0/0 | Low Pressure Switch | Closes at 90 PSI and above |
| I:0/1 | High Pressure Switch | Closes at 110 PSI and above |

### Physical Outputs

| Address | Tag | Description |
|---|---|---|
| O:0/0 | Pressure Pump | Raises receiver pressure |
| O:0/1 | Pressure Indicator Light | Indicates the low-pressure threshold has been reached |

### Internal Control Bits

| Address | Tag | Purpose |
|---|---|---|
| B3:0/0 | LOW_PRESSURE_SWITCH | Internal representation of low-pressure input |
| B3:0/1 | HIGH_PRESSURE_SWITCH | Internal representation of high-pressure input |
| B3:0/2 | PRESSURE_PUMP | Internal pump command |
| B3:0/3 | PRESSURE_IND_LIGHT | Internal pressure indicator command |

---

# 🧠 Problem-Solving Process

The most important part of this project was not simply getting the ladder logic to work.

It was learning how to take a written process description and systematically turn it into explicit control logic.

My understanding developed in stages:

1. Build a truth/state table from the written requirements
2. Use the table to derive explicit Boolean logic
3. Identify repeated input combinations
4. Determine whether previous process state matters
5. Introduce memory only where the process requires it
6. Test every required process condition
7. Review and simplify unnecessary retained state
8. Compare my implementation with the instructor's
9. Recognize the limitations of pure Boolean logic in a real physical environment
10. Add signal qualification and event-based control where appropriate

---

## Step 1 — Build the Truth / State Table

Rather than beginning directly with ladder logic, I first translated the required process sequence into a truth/state table.

| Process State | Low | High | Pump | Indicator |
|---|---:|---:|---:|---:|
| Below 90 PSI | 0 | 0 | ON | OFF |
| Rising above 90 PSI | 1 | 0 | ON | ON |
| At / above 110 PSI | 1 | 1 | OFF | ON |
| Falling below 110 PSI | 1 | 0 | OFF | ON |
| Falling below 90 PSI | 0 | 0 | ON | OFF |

This was my first major discovery.

The truth table gave me a repeatable way to translate a written process description into explicit logic.

Instead of immediately asking:

> "What ladder instruction should I use?"

I could first ask:

> "For this exact combination of inputs, what should every output be?"

That gave me a design process:

    Written Process Requirement
            ↓
    Identify Inputs
            ↓
    Identify Outputs
            ↓
    Enumerate Process States
            ↓
    Assign Explicit 0 / 1 Conditions
            ↓
    Translate Those Conditions Into Ladder Logic

The truth table therefore became more than a testing tool.

It became a way to derive the logic.

---

## Step 2 — Identify Repeated Input Combinations

The table contains repeated input combinations.

For example:

    LOW = 1
    HIGH = 0

appears twice.

At first, this looked like a duplicate state that might simply be combined.

However, the required pump output is different.

While pressure is rising:

    LOW = 1
    HIGH = 0
    PUMP = ON

Later, while pressure is falling:

    LOW = 1
    HIGH = 0
    PUMP = OFF

This means the current inputs alone are not enough to determine the pump state.

There is information missing from the current Boolean inputs.

That missing information is:

> What happened previously?

---

# 🧠 Discovering Memory / State Persistence

This repeated input combination was the key clue that the pump required memory.

If the PLC only evaluated:

    LOW = 1
    HIGH = 0

there would be no way to know whether the pump should currently be ON or OFF.

The answer depends on the process history.

Therefore:

    Pump State
    =
    Current Inputs
    +
    Previous Process State

This was how I discovered the need for state persistence.

I did not start with:

> "I should use a seal-in."

The truth table showed me why some form of memory was required.

A useful general rule became:

> If identical current inputs require different outputs depending on how the process arrived there, the controller must preserve some information about previous state.

---

## Step 3 — Remove True Duplicates

Not every repeated input combination represents a different state.

If the same input combination appears multiple times and requires the same output behavior, those rows can be treated as true duplicates.

However, if the same inputs require different outputs depending on previous process history, that distinction must remain.

That distinction helped reduce the process description into the minimum meaningful states before writing ladder logic.

---

# 🛠️ Control Strategy

## Digital I/O Mapping

Physical inputs are first mapped into internal B3 control bits.

This keeps the physical I/O layer separate from the control logic.

    Physical Input
          ↓
    Internal B3 Bit
          ↓
    Control Logic
          ↓
    Internal Output Bit
          ↓
    Physical Output

This structure makes the program easier to read, troubleshoot, test, and expand.

---

# Pressure Pump Control

The truth-table analysis showed that the pump requires persistent state.

The same input combination:

    LOW = 1
    HIGH = 0

must produce two different pump states depending on process history.

### Rising Pressure

    LOW = 1
    HIGH = 0
    PUMP = ON

The pump was already running before the low-pressure switch closed, so it must continue running.

### Falling Pressure

    LOW = 1
    HIGH = 0
    PUMP = OFF

The high-pressure switch previously stopped the pump, so the pump must remain stopped.

Because the current inputs cannot distinguish those two situations, the pump uses a seal-in / hysteresis pattern to preserve its operating state.

---

## Pump Sequence

### Pressure Below 90 PSI

    Low-pressure switch = OFF
    High-pressure switch = OFF
    Pump starts

The receiver pressure is low and the pump begins increasing pressure.

### Pressure Reaches 90 PSI

    Low-pressure switch = ON
    High-pressure switch = OFF

The original low-pressure start condition disappears.

However, the pump must continue running toward 110 PSI.

The pump's own control bit therefore maintains the pump through the seal-in path.

    Pump already ON
          ↓
    Pump contact becomes TRUE
          ↓
    Holding path remains complete
          ↓
    Pump continues running

### Pressure Reaches 110 PSI

    Low-pressure switch = ON
    High-pressure switch = ON
    Pump stops

The high-pressure condition breaks the pump holding path.

### Pressure Falls Below 110 PSI

    Low-pressure switch = ON
    High-pressure switch = OFF
    Pump remains OFF

This is the same Boolean input combination that previously existed while pressure was rising.

However, the pump remains OFF because the previous process state is different.

### Pressure Falls Below 90 PSI

    Low-pressure switch = OFF
    High-pressure switch = OFF
    Pump restarts

The process cycle begins again.

This produces the required hysteresis behavior without rapidly cycling the pump around a single pressure threshold.

---

# Pressure Indicator Light — Original Solution

I initially approached the indicator using the same state-table reasoning I used for the pump.

The required sequence was:

| Low | High | Indicator |
|---:|---:|---:|
| 0 | 0 | OFF |
| 1 | 0 | ON |
| 1 | 1 | ON |
| 1 | 0 | ON |
| 0 | 0 | OFF |

My first implementation used OTL / OTU retained-state logic.

### Light ON

When:

    LOW_PRESSURE_SWITCH = 1
    HIGH_PRESSURE_SWITCH = 0

the pressure indicator was latched ON.

### Light OFF

When:

    LOW_PRESSURE_SWITCH = 0
    HIGH_PRESSURE_SWITCH = 0

the pressure indicator was unlatched.

When both pressure switches were ON, neither instruction changed the light state, allowing the previous ON state to remain.

This implementation worked and reproduced the required test sequence.

---

# ♻️ First Refactor — Does the Indicator Actually Need Memory?

Although the latch/unlatch implementation worked, reviewing the truth table exposed another important lesson.

The pump and indicator initially looked similar because both were derived from the same state table.

However, they have fundamentally different state requirements.

## Pump

The same inputs:

    LOW = 1
    HIGH = 0

can require:

    PUMP = ON

or:

    PUMP = OFF

depending on process history.

Therefore:

    Pump State
    =
    Current Inputs
    +
    Previous State

Memory is required.

## Indicator

For the indicator:

    LOW = 0
        ↓
    LIGHT = OFF

    LOW = 1
        ↓
    LIGHT = ON

The required indicator state does not depend on whether pressure is rising or falling.

Therefore:

    Indicator State
    =
    Current Process Condition

The indicator does not require historical memory to satisfy the Boolean process requirement.

This meant my original OTL / OTU implementation was more complicated than necessary.

---

## Refactored Indicator Logic

The simpler implementation allows the indicator to directly follow the low-pressure condition through a normal OTE.

Conceptually:

    LOW_PRESSURE_SWITCH          PRESSURE_IND_LIGHT
    --------] [------------------------( )--------

The behavior becomes:

    LOW_PRESSURE_SWITCH = 1
            ↓
    Rung TRUE
            ↓
    Indicator ON

and:

    LOW_PRESSURE_SWITCH = 0
            ↓
    Rung FALSE
            ↓
    Indicator OFF

No separate de-energize instruction is required because the OTE is reevaluated every PLC scan.

---

## Why the Indicator Refactor Is Cleaner

The refactored version:

- Reduces two control rungs to one
- Removes unnecessary retained state
- Eliminates separate latch and unlatch instructions
- Directly represents the current physical process condition
- Reduces the number of states the programmer must mentally track
- Makes abnormal states easier to reason about
- Improves readability during troubleshooting

The original latch/unlatch implementation was still useful because it documented my actual problem-solving process.

It showed me that:

> A solution can work correctly and still contain unnecessary complexity.

---

# 🔒 Use Retained Latch Logic Only When the Process Actually Requires It

Another design principle I learned during this project was that I do not want to default to `OTL` / `OTU` simply because an output needs to remain ON.

An OTL / OTU pair answers a question like:

> "Did something happen that I need to remember after the original condition disappears?"

Conceptually:

    Set Event
        ↓
    OTL
        ↓
    State remains ON

and later:

    Reset Event
        ↓
    OTU
        ↓
    State returns OFF

That can be appropriate when retained event memory is actually required.

Examples include:

- faults that must remain recorded
- alarms requiring acknowledgment
- sequence states
- operator requests that must persist
- events that must remain remembered until an explicit reset

However, an output simply remaining ON does not automatically mean that OTL / OTU should be used.

The first question should be:

> Can the required output be completely determined from the current process state?

If YES:

    Current Process Condition
            ↓
    OTE
            ↓
    Output reflects current condition

If NO:

    Current Inputs
          +
    Previous Process State
          ↓
    Memory is required

Then the next question becomes:

> What is the clearest form of memory for this process?

Possible approaches may include:

- seal-in / holding logic
- sequence state bits
- state-machine logic
- OTL / OTU
- fault memory
- other intentional retained-state patterns

The decision process became:

    Can current inputs completely
    determine the required output?
                ↓
          YES / NO
          /       \
        YES        NO
         ↓          ↓
    Direct state   Memory
    logic / OTE    required
                      ↓
               Choose the most
               appropriate form
               of persistence

A key lesson became:

> Use memory because the process requires memory, not simply because a latch instruction is available.

---

# 🧪 Test Sequence & Verification

After completing my original implementation, I tested the program against every state required by the assignment.

| Step | Low | High | Pump | Indicator |
|---|---:|---:|---:|---:|
| Initial — below 90 PSI | 0 | 0 | ON | OFF |
| Low switch closes | 1 | 0 | ON | ON |
| High switch closes | 1 | 1 | OFF | ON |
| High switch opens | 1 | 0 | OFF | ON |
| Low switch opens | 0 | 0 | ON | OFF |

I verified each condition exactly as the instructions called it out.

Every required state produced the expected pump and indicator behavior.

This confirmed that the Boolean logic derived from the truth/state table was correct.

That validation was important because the next refactor was not caused by the original logic failing the assignment.

The original program worked.

The next lesson came from comparing a logically correct solution with a more industrial control architecture.

---

# 📸 Before & After

### Before — Initial Truth-Table / Boolean Implementation

<!-- Add original implementation screenshot here -->

### After — Refactored Real-World Control Architecture

<!-- Add refactored implementation screenshot here -->

---

# 🔍 Comparing My Solution to the Instructor's Solution

After completing and verifying my own solution, I reviewed the instructor's implementation rung by rung.

This comparison exposed another layer of PLC programming that my original truth-table approach had not considered.

The important lesson was not:

> "My program was wrong and the instructor's was right."

My program successfully satisfied every required process state.

Instead, the two implementations were solving the problem at different levels of abstraction.

My original implementation primarily answered:

> "Given these input states, what should the machine be doing?"

The instructor's implementation also considered:

> "Should I trust this physical input immediately?"

> "Has the condition existed long enough to be considered real?"

> "Is this a sustained condition or an event?"

> "Should this command happen continuously or only once?"

> "What changes the machine state?"

> "What holds the new state?"

> "What explicitly ends that state?"

---

# My Original Architecture

My original control architecture was comparatively direct.

    Pressure Switch State
            ↓
    Boolean Decision
            ↓
    Pump / Indicator State

For the pump:

    Low Pressure
        ↓
    Start Pump
        ↓
    Pump Holds Itself ON
        ↓
    High Pressure
        ↓
    Break Hold
        ↓
    Pump OFF

This correctly solved the state problem exposed by the truth table.

It also correctly implemented hysteresis.

However, the physical pressure-switch state was allowed to affect the control decision immediately.

---

# Instructor Architecture

The instructor inserted additional layers between the physical condition and the machine-state change.

The larger pattern was:

    Physical Input
        ↓
    TON
        ↓
    Timer DONE
        ↓
    ONS
        ↓
    Start / Stop Event
        ↓
    Seal-In / Hold Circuit
        ↓
    Machine State

This introduced two major concepts that were missing from my original implementation:

1. Signal qualification
2. Event generation

---

# ⏱️ Second Refactor — Signal Qualification

My truth table contained Boolean state.

It did not contain time.

For example:

    LOW = 0 for 20 milliseconds

and:

    LOW = 0 for 5 seconds

both appear identical in a truth table:

    LOW = 0

However, those two conditions may not deserve the same physical response.

A real pressure switch can be affected by:

- vibration
- pressure oscillation
- mechanical contact bounce
- fluid pulsation
- electrical noise
- temporary threshold crossings

My original architecture effectively assumed:

    Input changes
        ↓
    Logic evaluates
        ↓
    Machine reacts

The instructor's architecture instead used:

    Input changes
        ↓
    Detect condition
        ↓
    TON
        ↓
    Condition must remain valid
        ↓
    Timer DONE
        ↓
    Accept condition

The timer introduces the question:

> "Has this physical condition remained valid long enough that I should trust it?"

For example:

    Low-pressure condition appears
            ↓
    Start timer
            ↓
    Still low pressure?
       /             \
     NO               YES
     ↓                 ↓
    Reset            Continue
    timer             timing
                        ↓
                    Timer DONE
                        ↓
                 Accept condition

This showed me that logically correct Boolean state and trustworthy physical process state are not always the same thing.

---

# ⚡ Conditions vs. Events

The instructor's use of one-shots introduced another important distinction.

A condition may remain TRUE for many PLC scans.

For example:

> "Low pressure has existed for 5 seconds."

That is a condition.

However:

> "Start the pump."

is an event.

It only needs to happen once.

The instructor's architecture converted the sustained condition into a one-scan event:

    Qualified Condition
            ↓
    Timer .DN
            ↓
    ONS
            ↓
    One-Scan Event
            ↓
    Pump Start Trigger

This helped me distinguish:

> "This condition currently exists."

from:

> "This event just occurred."

The timer and one-shot therefore perform different jobs.

### TON

Answers:

> "Has the condition remained valid long enough to trust?"

### ONS

Answers:

> "Did this qualified condition just become actionable?"

---

# 🧱 Holding Machine State

Once the start command becomes a one-scan event, that event cannot remain TRUE to keep the pump running.

The machine therefore needs a holding mechanism.

The architecture becomes:

    Pump Start Event
            ↓
    Pump turns ON
            ↓
    Pump's own control bit becomes TRUE
            ↓
    Own contact maintains rung continuity
            ↓
    Pump remains ON

The one-shot disappears after one scan.

The pump remains ON because its own state maintains the holding path.

A separately generated stop event then breaks that path:

    High-pressure condition
            ↓
    Qualification timer
            ↓
    Timer DONE
            ↓
    ONS
            ↓
    Pump Interrupt
            ↓
    Break holding path
            ↓
    Pump OFF

This helped me recognize that the program was not simply a collection of instructions.

It had an architecture.

---

# 🧩 Rung-Level Architecture Pattern

The instructor's control structure can be grouped into larger patterns.

## Pump Start

    Pressure Condition
          ↓
    TON
          ↓
    INPUT QUALIFICATION
          ↓
    Timer DONE
          ↓
    ONS
          ↓
    START EVENT
          ↓
    HOLD CIRCUIT
          ↓
    Pump Running

## Pump Stop

    High-Pressure Condition
          ↓
    TON
          ↓
    INPUT QUALIFICATION
          ↓
    Timer DONE
          ↓
    ONS
          ↓
    STOP / INTERRUPT EVENT
          ↓
    BREAK HOLD
          ↓
    Pump OFF

## Indicator Start

    90 PSI Condition
          ↓
    TON
          ↓
    INPUT QUALIFICATION
          ↓
    Timer DONE
          ↓
    ONS
          ↓
    INDICATOR ON EVENT
          ↓
    HOLD INDICATOR

## Indicator Stop

    90 PSI Condition Removed
          ↓
    TON
          ↓
    INPUT QUALIFICATION
          ↓
    Timer DONE
          ↓
    ONS
          ↓
    INDICATOR OFF EVENT
          ↓
    BREAK INDICATOR HOLD

---

# 🧠 Pattern Recognition

I can now recognize the individual ladder instructions as parts of larger control patterns.

    Sensor + Timer
    =
    Input Qualification

    Timer DONE + One-Shot
    =
    Event Generator

    Start Event + Own Contact
    =
    Hold / Seal-In

    Stop Event + XIO
    =
    Interrupt the Hold

The larger pattern becomes:

> **Detect → Qualify → Trigger → Hold → Interrupt**

---

# 🔄 What the Truth Table Did — and Did Not — Tell Me

The truth table was not wrong.

It was one of the most useful tools in the project.

It allowed me to:

- derive explicit logic from written requirements
- identify every required process state
- identify repeated input combinations
- identify true duplicates
- discover where process history mattered
- derive the need for memory
- understand hysteresis
- determine when retained memory was unnecessary
- verify that the completed ladder logic satisfied every required condition

However, the truth table could not answer:

- What if a pressure switch flickers?
- What if vibration causes contact bounce?
- What if pressure crosses the threshold for only a few milliseconds?
- Should every input transition immediately change the machine?
- Is this signal describing a sustained condition or a discrete event?
- Should a command happen once or continuously?
- What should preserve the machine state after the event disappears?
- What explicitly ends that state?

Those questions only became visible when I moved from thinking about Boolean logic alone to thinking about a PLC connected to physical equipment.

---

# 🧠 State, Memory, Time, and Events

This project helped me separate four concepts that initially looked like the same problem.

## State

> What is true right now?

Example:

    LOW = 1

## Memory

> Does the correct output depend on something that happened previously?

Example:

    LOW = 1
    HIGH = 0

may require different pump outputs depending on process history.

## Time

> Has the physical condition remained valid long enough to trust?

Example:

    Low pressure
        ↓
    TON
        ↓
    5 seconds
        ↓
    Qualified low pressure

## Event

> Did something meaningful just happen that should cause a state transition?

Example:

    Qualified low pressure
        ↓
    ONS
        ↓
    Pump Start Event

These are separate questions.

Recognizing that distinction changed how I approached the ladder logic.

---

# 🧠 Concepts Demonstrated

This project demonstrates:

- Truth-table development
- State-table analysis
- Translating written process requirements into explicit logic
- Identifying duplicate states
- Identifying states requiring process history
- Persistent-state reasoning
- Hysteresis
- Seal-in / holding circuits
- Digital input mapping
- Digital output mapping
- Internal control bits
- XIC instructions
- XIO instructions
- OTE instructions
- OTL / OTU retained-state logic
- Determining when OTL / OTU is unnecessary
- PLC scan behavior
- TON timers
- Input qualification
- Timer `.DN` bits
- ONS / one-shot event generation
- Condition vs. event reasoning
- Start / stop event architecture
- Interrupting hold circuits
- Separation of physical I/O and control logic
- Testing against defined process states
- Reviewing working logic for unnecessary complexity
- Comparing multiple valid control architectures
- Refactoring ladder logic for readability and physical robustness

---

# 📁 Program Structure

    MAIN
    │
    ├── DIGITAL IO
    │   │
    │   ├── Physical Inputs
    │   │       ↓
    │   │   Internal Control Bits
    │   │
    │   └── Internal Output Bits
    │           ↓
    │       Physical Outputs
    │
    └── CONTROLS
        │
        ├── Pressure Pump
        │   ├── Start condition
        │   ├── State persistence / seal-in
        │   └── Stop condition
        │
        └── Pressure Indicator
            └── Current pressure-state indication

The refactored architecture adds another conceptual layer:

    FIELD INPUT
        ↓
    DETECT
        ↓
    QUALIFY
        ↓
    GENERATE EVENT
        ↓
    CHANGE STATE
        ↓
    HOLD STATE
        ↓
    INTERRUPT STATE
        ↓
    PHYSICAL OUTPUT

---

# 💡 Key Takeaway

My approach to this project evolved significantly.

I started with:

    Process Description
            ↓
    Truth / State Table
            ↓
    Explicit Boolean Logic

The truth table gave me a systematic way to derive control logic instead of guessing at ladder instructions.

Then I discovered repeated states:

    LOW = 1
    HIGH = 0
    PUMP = ON

and later:

    LOW = 1
    HIGH = 0
    PUMP = OFF

That proved that the current inputs were not sufficient.

The process required memory.

That discovery led to:

    Current Inputs
          +
    Previous State
          ↓
    Correct Machine State

I then discovered that memory should not automatically be applied everywhere.

The pump required historical state.

The indicator, in my original Boolean implementation, did not.

That led to another rule:

> Use memory only when the process requires information to persist beyond the condition that created it.

After building my program, I verified every condition required by the assignment.

The program worked.

The pump and indicator entered every required state.

Only after that did I compare my solution with the instructor's implementation.

That comparison exposed the next layer.

My truth table answered:

> "What should the machine do?"

The instructor's architecture forced me to also ask:

> "When should I trust the physical input?"

> "Is this a condition or an event?"

> "Should the action happen once or continuously?"

> "What changes the machine state?"

> "What preserves that state?"

> "What explicitly ends it?"

My problem-solving process therefore became:

    Process Description
            ↓
    Truth / State Table
            ↓
    Derive Explicit Logic
            ↓
    Identify Repeated States
            ↓
    Determine Whether History Matters
            ↓
    Add Memory Only Where Required
            ↓
    Build Functional Ladder Logic
            ↓
    Verify Every Required Condition
            ↓
    Refactor Unnecessary Memory
            ↓
    Consider Real-World Input Behavior
            ↓
    Qualify Physical Conditions
            ↓
    Separate Conditions From Events
            ↓
    Generate Intentional Start / Stop Events
            ↓
    Hold Machine State
            ↓
    Interrupt State Intentionally

The biggest lesson from the project was not simply learning individual PLC instructions.

It was learning how to determine **why** each type of logic is needed.

The truth table taught me how to derive explicit logic.

Repeated states taught me when process memory must persist.

Refactoring taught me when memory should not persist.

Comparing my implementation with the instructor's taught me that correct Boolean logic is only one layer of industrial control.

A real PLC is connected to physical equipment whose signals exist over time and may not transition perfectly.

The larger design process I now recognize is:

> **Derive the State → Determine Memory → Detect → Qualify → Trigger → Hold → Interrupt**

That progression moved this project from simply making the ladder logic satisfy the assignment toward understanding how control logic represents and manages a real physical process.



### Before — Initial Truth-Table / Boolean Implementation
<img width="1212" height="341" alt="Screenshot 2026-09-16 at 5 19 49 PM" src="https://github.com/user-attachments/assets/0a73f6a6-3074-4f25-a098-79eb75e5e8db" />
<img width="1208" height="518" alt="Screenshot 2026-09-16 at 5 19 57 PM" src="https://github.com/user-attachments/assets/78052336-061a-46fc-ab9b-9f2872491337" />
<img width="1203" height="547" alt="Screenshot 2026-09-16 at 5 20 03 PM" src="https://github.com/user-attachments/assets/b8172a69-41de-4528-ad5e-9400b39b7b7c" />

### After — Refactored Real-World Control Architecture


