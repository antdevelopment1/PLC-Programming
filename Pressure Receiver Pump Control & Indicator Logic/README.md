# Pressure Receiver Pump Control & Indicator Logic

A PLC programming exercise built in **RSLogix Micro Starter Lite** to practice digital I/O mapping, pressure-based pump control, hysteresis, retained-state logic, truth-table analysis, and ladder-logic refactoring.

## 🔍 Overview

This project controls pressure inside a receiver using two pressure switches and a single pump.

The process uses:

- A **low-pressure switch** that closes at 90 PSI and above
- A **high-pressure switch** that closes at 110 PSI and above
- A **pressure pump** used to increase receiver pressure
- A **pressure indicator light** that illuminates once the low-pressure threshold is reached

The pump must:

- Start when pressure is below 90 PSI
- Continue running after the low-pressure switch closes
- Stop when pressure reaches 110 PSI
- Remain stopped while pressure falls back below 110 PSI
- Restart only after pressure falls below 90 PSI

This creates a basic **hysteresis control pattern**, using separate start and stop thresholds instead of cycling the pump around a single pressure point.

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
| `I:0/0` | Low Pressure Switch | Closes at 90 PSI and above |
| `I:0/1` | High Pressure Switch | Closes at 110 PSI and above |

### Physical Outputs

| Address | Tag | Description |
|---|---|---|
| `O:0/0` | Pressure Pump | Raises receiver pressure |
| `O:0/1` | Pressure Indicator Light | Indicates the low-pressure threshold has been reached |

### Internal Control Bits

| Address | Tag | Purpose |
|---|---|---|
| `B3:0/0` | `LOW_PRESSURE_SWITCH` | Internal representation of the low-pressure input |
| `B3:0/1` | `HIGH_PRESSURE_SWITCH` | Internal representation of the high-pressure input |
| `B3:0/2` | `PRESSURE_PUMP` | Internal pump command |
| `B3:0/3` | `PRESSURE_IND_LIGHT` | Internal pressure indicator command |

---

## 🧠 Problem-Solving Process

Rather than beginning directly with ladder logic, I first translated the required test sequence into a **truth/state table**.

This made it easier to separate three questions:

1. What are the current input states?
2. What should each output be in that state?
3. Does the required output depend only on the current inputs, or does it also depend on what happened previously?

### Step 1 — Build the State Table

The required sequence can be represented as:

| Process State | Low | High | Pump | Indicator |
|---|---:|---:|---:|---:|
| Below 90 PSI | 0 | 0 | ON | OFF |
| Rising above 90 PSI | 1 | 0 | ON | ON |
| At / above 110 PSI | 1 | 1 | OFF | ON |
| Falling below 110 PSI | 1 | 0 | OFF | ON |
| Falling below 90 PSI | 0 | 0 | ON | OFF |

### Step 2 — Identify Repeated Input Combinations

The table contains repeated input combinations.

For example:

```text
LOW = 1
HIGH = 0
```

appears twice.

However, the required pump output is different:

```text
Pressure rising:
LOW = 1
HIGH = 0
PUMP = ON
```

but later:

```text
Pressure falling:
LOW = 1
HIGH = 0
PUMP = OFF
```

That means the current inputs alone are **not enough to determine the pump state**.

The PLC must retain some information about what happened previously.

This was the key clue that the pump required **memory / state persistence**.

### Step 3 — Remove True Duplicates

Where an input combination appeared more than once and required the **same output behavior**, I treated those rows as duplicates rather than building separate logic for each occurrence.

Where the same inputs required **different outputs depending on the previous process state**, I kept that distinction because it indicated that memory was required.

This reduced the process description into the minimum set of meaningful states before writing ladder logic.

---

## 🛠️ Control Strategy & Key Rungs

### Digital I/O Mapping

Physical inputs are first mapped into internal `B3` control bits.

This keeps the physical I/O layer separate from the control logic and allows the controls file to operate using descriptive internal tags.

The internal output commands are then mapped back to the physical outputs.

```text
Physical Input
      ↓
Internal B3 Bit
      ↓
Control Logic
      ↓
Internal Output Bit
      ↓
Physical Output
```

This structure makes the program easier to read, troubleshoot, and expand.

---

## Pressure Pump Control

The truth-table analysis showed that the pump requires **persistent state**.

The same input combination:

```text
LOW = 1
HIGH = 0
```

must produce two different results depending on where the process came from.

### Rising Pressure

```text
LOW = 1
HIGH = 0
PUMP = ON
```

The pump was already running before the low-pressure switch closed, so it must continue running.

### Falling Pressure

```text
LOW = 1
HIGH = 0
PUMP = OFF
```

The high-pressure switch previously stopped the pump, so the pump must remain stopped until pressure falls below the low threshold.

Because the current inputs cannot distinguish those two situations, the pump logic uses a **seal-in / hysteresis pattern** to preserve the previous operating state.

### Pump Sequence

When pressure is below 90 PSI:

- Low-pressure switch = OFF
- High-pressure switch = OFF
- Pump starts

When pressure rises to 90 PSI and above:

- Low-pressure switch closes
- Pump remains energized through its own control bit

When pressure reaches 110 PSI:

- High-pressure switch closes
- Pump stops

When pressure falls below 110 PSI:

- High-pressure switch opens
- Pump remains stopped

The pump does not restart until pressure falls below 90 PSI and the low-pressure switch opens.

This produces the required hysteresis behavior without rapidly cycling the pump around a single pressure threshold.

---

## Pressure Indicator Light — Original Solution

I initially approached the indicator light using the **same state-table process** I used for the pump.

The required sequence was:

| Low | High | Indicator |
|---:|---:|---:|
| 0 | 0 | OFF |
| 1 | 0 | ON |
| 1 | 1 | ON |
| 1 | 0 | ON |
| 0 | 0 | OFF |

My first implementation used **OTL / OTU retained-state logic**.

### Light ON

When:

```text
LOW_PRESSURE_SWITCH = 1
HIGH_PRESSURE_SWITCH = 0
```

`PRESSURE_IND_LIGHT` is latched ON.

### Light OFF

When:

```text
LOW_PRESSURE_SWITCH = 0
HIGH_PRESSURE_SWITCH = 0
```

`PRESSURE_IND_LIGHT` is unlatched.

When both pressure switches are ON, neither instruction changes the light state, allowing the previously latched ON state to persist.

This implementation reproduces the required test sequence.

---

## 🔎 Reviewing the Indicator Solution

Although the latch/unlatch implementation works, I did not prefer the result because it introduced **persistent memory into an output that did not actually require memory**.

The pump and indicator initially looked similar because both were being derived from the same state table.

The important difference became clearer after reviewing the completed logic.

### Pump

The same inputs can require different outputs:

```text
LOW = 1
HIGH = 0
```

can mean:

```text
PUMP = ON
```

or:

```text
PUMP = OFF
```

depending on previous process history.

Therefore:

> **Pump state depends on current inputs + previous state.**

Memory is required.

### Indicator

For the indicator:

```text
LOW = 0 → LIGHT = OFF
LOW = 1 → LIGHT = ON
```

The desired light state does not actually depend on how the process reached that condition.

Therefore:

> **Indicator state depends only on the current process condition.**

Memory is not required.

This was an important distinction that became apparent through reviewing and refactoring the original working solution.

---

## ♻️ Refactor Opportunity — Indicator Light

A simpler implementation would remove the latch/unlatch pair and allow the indicator to directly follow the low-pressure condition through a normal **OTE**.

Conceptually:

```text
LOW_PRESSURE_SWITCH          PRESSURE_IND_LIGHT
--------] [------------------------( )--------
```

The behavior then becomes:

```text
LOW_PRESSURE_SWITCH = 1
        ↓
Rung TRUE
        ↓
Indicator ON
```

and:

```text
LOW_PRESSURE_SWITCH = 0
        ↓
Rung FALSE
        ↓
Indicator OFF automatically
```

No separate de-energize instruction is required because an OTE is reevaluated every PLC scan.

### Why the Refactor Is Cleaner

The refactored version:

- Reduces two control rungs to one
- Removes unnecessary retained state
- Eliminates separate latch and unlatch instructions
- Directly represents the physical process condition
- Makes abnormal input combinations easier to reason about
- Reduces the number of states the programmer must mentally track
- Improves readability during troubleshooting

The original latch/unlatch implementation is intentionally retained in the project because it documents the actual problem-solving process and demonstrates why a working solution can still be improved.

---

## 💡 State vs. Memory

One of the main lessons from this project was learning to distinguish between **state-based logic** and **memory-based logic**.

### OTE — Current State

An OTE answers:

> **"Should this output be ON right now?"**

```text
Rung TRUE  → Output ON
Rung FALSE → Output OFF
```

This is appropriate when the output directly represents a current process condition.

### OTL / OTU — Retained Memory

A latch/unlatch pair answers:

> **"Did an event occur that I need to remember until another event clears it?"**

```text
Set condition
      ↓
OTL
      ↓
State remains ON

Reset condition
      ↓
OTU
      ↓
State returns OFF
```

This is useful for conditions such as:

- Fault memory
- Alarm acknowledgment
- Sequence states
- Operator requests that must persist
- Events that must remain recorded after the triggering condition disappears

The presence of a repeated input combination with **different required outputs** is one indication that some form of persistent state may be necessary.

---

## 🧪 Test Sequence

The finished program was evaluated using the required pressure-switch sequence:

| Step | Low | High | Pump | Indicator |
|---|---:|---:|---:|---:|
| Initial | 0 | 0 | ON | OFF |
| Low switch closes | 1 | 0 | ON | ON |
| High switch closes | 1 | 1 | OFF | ON |
| High switch opens | 1 | 0 | OFF | ON |
| Low switch opens | 0 | 0 | ON | OFF |

The pump and indicator logic were designed to satisfy each required process state.

---

## 🧠 Concepts Demonstrated

- Truth-table development
- State-table analysis
- Identifying duplicate states
- Identifying states that require process memory
- Persistent-state reasoning
- Digital input mapping
- Digital output mapping
- Internal control bits
- XIC and XIO instructions
- OTE instructions
- OTL / OTU retained-state logic
- Seal-in logic
- Hysteresis control
- PLC scan behavior
- Separation of physical I/O and control logic
- Testing against defined process states
- Reviewing working logic for unnecessary complexity
- Refactoring ladder logic for readability

---

## 📁 Program Structure

```text
MAIN
│
├── DIGITAL IO
│   ├── Physical inputs → internal control bits
│   └── Internal output bits → physical outputs
│
└── CONTROLS
    ├── Pressure pump hysteresis / memory logic
    └── Pressure indicator latch / unlatch logic
```

---

## 💡 Key Takeaway

My approach to this project was:

```text
Process Description
        ↓
Truth / State Table
        ↓
Identify Repeated Input States
        ↓
Remove True Duplicates
        ↓
Identify Where Previous State Matters
        ↓
Build Ladder Logic
        ↓
Test
        ↓
Review / Refactor
```

The most important discovery was that **identical current inputs do not always mean identical required behavior**.

For the pump, the repeated `LOW = 1 / HIGH = 0` state required different pump outputs depending on whether pressure was rising or falling. That revealed the need for persistent state and led to the seal-in / hysteresis solution.

I initially applied a similar memory-oriented approach to the indicator light, which produced a working latch/unlatch implementation. After reviewing the result, however, I recognized that the light did not need to remember anything — it only needed to represent the current state of the low-pressure switch.

That created a clear refactoring opportunity:

> **Use memory only when the process requires memory.**

The project therefore demonstrates not only how I arrived at a working PLC program, but also how I evaluated my own solution afterward and identified a simpler implementation.

<img width="1212" height="341" alt="Screenshot 2026-09-16 at 5 19 49 PM" src="https://github.com/user-attachments/assets/d3fae7b4-2add-41bf-956b-8432fda18d02" />
<img width="1208" height="518" alt="Screenshot 2026-09-16 at 5 19 57 PM" src="https://github.com/user-attachments/assets/56c5691a-2467-42a5-9484-d7ad29f81426" />
<img width="1203" height="547" alt="Screenshot 2026-09-16 at 5 20 03 PM" src="https://github.com/user-attachments/assets/8503b30e-0842-4b78-8e4e-ff7f60ddb5f9" />

