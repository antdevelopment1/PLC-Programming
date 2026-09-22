# Industrial Mixer Manual & Auto Permissive Logic

A PLC programming exercise built in **CODESYS** to practice Boolean logic, permissives, seal-in logic, manual/automatic operating modes, PLC scan behavior, and ladder-logic refactoring.

## 🔍 Overview

This project controls an industrial mixer with two possible operating modes.

### Manual Operation

The mixer may start when:

- The **Start button** is pressed
- The **lid is closed**
- The **operator is ready**

Once the mixer starts, it must remain running after the Start button is released.

### Automatic Operation

When **Auto Mode** is active:

- The mixer may start automatically when the **lid is closed**
- The **Operator Ready** condition is not required

### Stop Conditions

Regardless of operating mode, the mixer must stop when:

- The **Stop button** is pressed
- The **lid opens**

---

## ⚙️ Platform & Tools

- **Programming Software:** CODESYS Development System
- **Language:** Ladder Diagram (LD)
- **Programming Style:** Symbolic variables / Boolean logic
- **Primary Concepts:** Permissives, seal-in logic, parallel branches, manual/auto control

---

## 🗺️ Variables

| Variable | Type | Purpose |
|---|---|---|
| `xStartBtn` | `BOOL` | Manual Start command |
| `xStopBtn` | `BOOL` | Stops mixer operation |
| `xLidClosed` | `BOOL` | Confirms mixer lid is closed |
| `xOperatorRdy` | `BOOL` | Confirms operator is ready for manual operation |
| `xAutoModeBtn` | `BOOL` | Enables automatic operation |
| `xRunMixer` | `BOOL` | Mixer run command |

---

## 🧠 Translating the Requirements

I did not begin this project with a truth table.

Because of my previous PLC programming experience, I first translated the process description directly into the major control conditions.

The requirements naturally separated into two operating paths.

### Manual Path

```text
START
AND
OPERATOR READY
AND
LID CLOSED
