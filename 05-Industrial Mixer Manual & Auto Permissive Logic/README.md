<img width="934" height="384" alt="Screenshot 2026-09-22 at 1 05 05 AM" src="https://github.com/user-attachments/assets/17a95f46-9a9f-4099-9ca4-aabf5fce013c" />
<!-- Folder Name: 05-Industrial Mixer Manual & Auto Permissive Logic -->

# Industrial Mixer Manual & Auto Permissive Logic

A **CODESYS Ladder Diagram** exercise focused on translating process requirements into PLC logic, implementing manual and automatic operating modes, using seal-in logic, and refactoring an initial solution after identifying a multiple-output-write issue.

<img width="943" height="386" alt="Screenshot 2026-09-22 at 1 05 30 AM" src="https://github.com/user-attachments/assets/39de0080-b7a0-4e34-926e-6b11f6ee29ad" />


---

## 🔍 Project Requirements

The mixer has two operating modes.

### Manual Mode

The mixer should start when:

- The **Start button** is pressed
- The **lid is closed**
- The **operator is ready**

Once started, the mixer should continue running after the Start button is released.

### Automatic Mode

When **Auto Mode** is active:

- The mixer can run automatically when the **lid is closed**
- **Operator Ready is not required**

### Stop Conditions

Regardless of operating mode, the mixer must stop when:

- The **Stop button is pressed**
- The **lid opens**

---

## ⚙️ Platform

- **Software:** CODESYS Development System
- **Language:** Ladder Diagram (LD)
- **Variable Type:** BOOL
- **Primary Concepts:** Boolean logic, permissives, seal-in logic, manual/auto modes, PLC scan behavior, refactoring

---

## 🗺️ Variables

| Variable | Type | Purpose |
|---|---|---|
| `xStartBtn` | `BOOL` | Manual Start command |
| `xStopBtn` | `BOOL` | Stop command |
| `xLidClosed` | `BOOL` | Confirms the mixer lid is closed |
| `xOperatorRdy` | `BOOL` | Confirms the operator is ready |
| `xAutoModeBtn` | `BOOL` | Enables automatic operation |
| `xRunMixer` | `BOOL` | Mixer run command |

---

# 🧠 My Initial Approach

I completed the first version quickly without creating a truth table.

Based on previous PLC programming experience, I read the process description and translated the requirements directly into ladder logic.

I initially saw the problem as two separate operating modes:

### Manual

```text
Start
AND
Operator Ready
AND
Lid Closed
```

with a seal-in around the Start button so the mixer would continue running after Start was released.

### Automatic

```text
Auto Mode
AND
Lid Closed
```

Since Auto Mode specifically does not require Operator Ready, I treated it as a separate control path.

---

# 🛠️ Initial Implementation

My first solution used two separate rungs.

## Rung 1 — Manual Mode

Conceptually:

```text
      +----[ xStartBtn ]----+
      |                     |
------+                     +----[/ xStopBtn ]----[ xLidClosed ]----[ xOperatorRdy ]----( xRunMixer )
      |                     |
      +----[ xRunMixer ]----+
```

This rung included:

- Start command
- Run seal-in
- Stop condition
- Lid Closed permissive
- Operator Ready permissive

## Rung 2 — Automatic Mode

```text
----[/ xStopBtn ]----[ xAutoModeBtn ]----[ xLidClosed ]--------------------( xRunMixer )
```

This allowed the mixer to run automatically when:

```text
Auto Mode = TRUE
AND
Lid Closed = TRUE
AND
Stop = FALSE
```

Operator Ready was intentionally not included in the Auto rung.

---

# 🔎 Reviewing the First Version

The first version represented most of the process requirements correctly.

I had identified:

- Manual operation
- Automatic operation
- Start logic
- Seal-in logic
- Stop logic
- Lid permissive
- Operator Ready
- Auto bypass of Operator Ready

However, reviewing the program exposed a structural issue.

---

# ⚠️ Problem — Two Rungs Writing to the Same Output

Both rungs ended with the same normal output coil:

```text
xRunMixer
```

Conceptually:

```text
Manual Rung --------------------( xRunMixer )

Auto Rung ----------------------( xRunMixer )
```

Because the PLC scans the program sequentially, each rung can write a new value to the same variable during the same scan.

For example, during Manual operation:

```text
Manual rung = TRUE

xRunMixer = TRUE
```

But if Auto Mode is not active:

```text
Auto rung = FALSE

xRunMixer = FALSE
```

Since the Auto rung is evaluated later, it can overwrite the TRUE state that was written by the Manual rung.

So even though my two individual operating-mode conditions made sense, the structure of the program created a multiple-write problem.

---

# 💡 Refactoring the Logic

Instead of thinking:

```text
Manual Logic → RunMixer

Auto Logic → RunMixer
```

I refactored the program so Manual and Auto became two alternative paths leading to **one final `xRunMixer` coil**.

Conceptually:

```text
                 +---- Manual Logic ----+
                 |                      |
Shared Conditions                        +----( xRunMixer )
                 |                      |
                 +---- Auto Logic ------+
```

The two modes are still separate logically, but they no longer write independently to the same output.

---

# 🧠 Identifying the Shared Conditions

Looking at the requirements again, both modes require:

```text
Stop NOT pressed
AND
Lid Closed
```

So instead of repeating those conditions inside both operating paths, they can be placed before the Manual/Auto branch.

That gives the general structure:

```text
NOT Stop
AND
Lid Closed
AND
(
    Manual Condition
    OR
    Auto Condition
)
```

---

# 🔧 Manual Branch

The Manual requirements are:

```text
Operator Ready
AND
Start
```

but the mixer must remain running after the Start button is released.

That means the Start command needs a seal-in:

```text
Start
OR
RunMixer
```

Operator Ready remains part of the Manual operating condition.

So the Manual branch becomes:

```text
Operator Ready
AND
(
    Start
    OR
    RunMixer
)
```

---

# 🤖 Automatic Branch

The Auto requirement is much simpler:

```text
Auto Mode
```

because `Lid Closed` and `Stop` are already handled as shared conditions before the branch.

Operator Ready is intentionally not required.

---

# ♻️ Final Refactored Logic

The final Boolean expression becomes:

```text
NOT Stop
AND
Lid Closed
AND
(
    Auto Mode
    OR
    (
        Operator Ready
        AND
        (
            Start
            OR
            RunMixer
        )
    )
)
```

---

## Final Ladder Structure

Conceptually:

```text
                                      +----[ xAutoModeBtn ]----------------------+
                                      |                                          |
----[/ xStopBtn ]----[ xLidClosed ]--+                                          +----( xRunMixer )
                                      |                                          |
                                      +----[ xOperatorRdy ]----+----[ xStartBtn ]+
                                                              |
                                                              +----[ xRunMixer ]+
```

Now there is only:

```text
ONE xRunMixer coil
```

and Manual and Auto are simply different logical paths that can energize it.

---

# 🧪 How the Final Logic Works

## Manual Start

Conditions:

```text
xStopBtn = FALSE
xLidClosed = TRUE
xOperatorRdy = TRUE
xStartBtn = TRUE
xAutoModeBtn = FALSE
```

Result:

```text
xRunMixer = TRUE
```

---

## Manual Seal-In

After the mixer starts:

```text
xStartBtn = FALSE
xRunMixer = TRUE
xOperatorRdy = TRUE
xLidClosed = TRUE
xStopBtn = FALSE
```

The `xRunMixer` contact provides the holding path.

Result:

```text
xRunMixer remains TRUE
```

The operator can release the Start button without stopping the mixer.

---

## Manual Start Without Operator Ready

```text
xStartBtn = TRUE
xOperatorRdy = FALSE
xAutoModeBtn = FALSE
```

Result:

```text
xRunMixer = FALSE
```

The manual path is incomplete.

---

## Auto Mode

Conditions:

```text
xAutoModeBtn = TRUE
xLidClosed = TRUE
xStopBtn = FALSE
```

Operator Ready can be:

```text
TRUE
or
FALSE
```

Result:

```text
xRunMixer = TRUE
```

because Operator Ready is not part of the Auto path.

---

## Stop Button Pressed

If:

```text
xStopBtn = TRUE
```

the shared Stop contact opens.

Result:

```text
xRunMixer = FALSE
```

regardless of which operating mode was active.

---

## Lid Opens

If:

```text
xLidClosed = FALSE
```

the shared Lid Closed permissive becomes false.

Result:

```text
xRunMixer = FALSE
```

regardless of Manual or Auto operation.

---

# 🧪 Functional Test Cases

| Test | Start | Stop | Lid Closed | Operator Ready | Auto | Expected Mixer |
|---|---:|---:|---:|---:|---:|---|
| Manual Start | 1 | 0 | 1 | 1 | 0 | ON |
| Start Released After Running | 0 | 0 | 1 | 1 | 0 | ON |
| Manual Without Operator Ready | 1 | 0 | 1 | 0 | 0 | OFF |
| Auto With Operator Ready | 0 | 0 | 1 | 1 | 1 | ON |
| Auto Without Operator Ready | 0 | 0 | 1 | 0 | 1 | ON |
| Stop Pressed | X | 1 | 1 | X | X | OFF |
| Lid Open | X | 0 | 0 | X | X | OFF |

`X` means the value does not affect the expected result.

---

# 🧠 What I Learned

The biggest lesson from this project was not how to create a seal-in circuit.

I was already comfortable identifying the need for:

```text
Start
OR
Run
```

The more important lesson was recognizing that two valid pieces of logic can still create a bad overall program structure when they independently write to the same output.

My original thought process was:

```text
There are two modes.

Manual mode gets a rung.

Auto mode gets a rung.
```

That seemed reasonable when looking only at the requirements.

The problem became clear when considering the PLC scan:

```text
Rung 1 writes xRunMixer
        ↓
Rung 2 writes xRunMixer again
```

Refactoring the program changed my thinking from:

```text
Two modes = two output rungs
```

to:

```text
Two modes = two possible paths to one output command
```

That produced a cleaner structure:

```text
                +---- Manual ----+
                |                |
Stop + Lid -----+                +---- RunMixer
                |                |
                +---- Auto ------+
```

---

# 🔍 Another Important Design Decision

I also reviewed the wording around `Operator Ready`.

The specification explicitly says the mixer:

```text
should keep running even after the Start button is released
```

It does not explicitly state that Operator Ready may be removed after startup.

Because of that, I kept Operator Ready as a continuing condition of the Manual path:

```text
OperatorReady
AND
(
    Start
    OR
    RunMixer
)
```

If the specification instead said:

```text
Operator Ready is only required to initiate the cycle
```

then the logic could be designed differently.

That distinction reinforced the importance of programming from the actual process specification instead of assuming behavior that was not defined.

---

# 🧠 Concepts Demonstrated

This project demonstrates:

- CODESYS programming
- Ladder Diagram
- BOOL variables
- Symbolic addressing
- Boolean AND logic
- Boolean OR logic
- Parallel ladder branches
- Manual / Auto control
- Permissive logic
- Seal-in / holding logic
- Shared operating conditions
- PLC scan behavior
- Multiple writes to the same variable
- Translating written requirements into ladder logic
- Reviewing an initial implementation
- Refactoring PLC logic
- Reducing duplicated logic
- Designing one final output command from multiple operating modes

---

# 📁 Program Structure

```text
PLC_PRG
│
└── Industrial Mixer Control
    │
    ├── Shared Conditions
    │   ├── Stop
    │   └── Lid Closed
    │
    ├── Operating Modes
    │   │
    │   ├── Manual
    │   │   ├── Operator Ready
    │   │   ├── Start
    │   │   └── RunMixer Seal-In
    │   │
    │   └── Auto
    │       └── Auto Mode
    │
    └── xRunMixer
```
---

# 💡 Key Takeaway

I built the initial solution quickly by translating the process description directly into ladder logic rather than creating a truth table first.

My previous PLC experience helped me identify the major conditions and the need for seal-in logic almost immediately.

The first implementation captured most of the required process behavior, but reviewing it exposed an important design problem:

```text
Manual Logic → RunMixer

Auto Logic → RunMixer
```

Both modes were independently writing to the same normal output.

The refactored design became:

```text
                +→ Manual Logic →+
Shared Conditions                 +→ RunMixer
                +→ Auto Logic   →+
```

The result is a program that is easier to read, easier to troubleshoot, avoids competing writes to the same output, and more clearly represents the relationship between the two operating modes.

The project reinforced an important PLC programming principle:

> **Separate operating modes do not necessarily require separate output coils. They can instead be separate logical paths that contribute to one final output command.**
