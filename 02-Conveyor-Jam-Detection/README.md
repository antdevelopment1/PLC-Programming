# Conveyor Jam Detection

## Overview

This project is a PLC ladder logic exercise built in **RSLogix 500**.

The goal is to control a conveyor with Start/Stop operation, safety permissives, photoeye-based jam detection, fault latching, and controlled fault recovery.

The exercise focuses on designing machine behavior over time rather than simply turning an output on and off.

---

## Task

Program a conveyor that:

- Starts from a momentary Start pushbutton
- Continues running after the Start button is released
- Stops when the Stop pushbutton is pressed
- Requires the E-stop circuit to be healthy
- Requires the machine guard to be closed
- Detects a box blocking the photoeye for 5 seconds
- Latches a jam fault when the 5-second timer completes
- Stops the conveyor when a jam fault occurs
- Turns on a fault indicator during a jam fault
- Prevents the fault from being reset while the photoeye is still blocked
- Requires a manual reset after the obstruction is removed
- Prevents the conveyor from automatically restarting after the fault is reset
- Requires the operator to press Start again before operation resumes

---

## Inputs

| Input | Internal Tag | Description |
|---|---|---|
| `I:0/0` | `START_PB` | Momentary conveyor Start pushbutton |
| `I:0/1` | `STOP_PB` | Conveyor Stop pushbutton |
| `I:0/2` | `RESET_PB` | Jam fault Reset pushbutton |
| `I:0/3` | `ESTOP_OK` | Indicates the E-stop circuit is healthy |
| `I:0/4` | `GUARD_OK` | Indicates the machine guard is closed |
| `I:0/5` | `PE_BOX_BLOCK` | Photoeye detects a box blocking the conveyor |

---

## Internal Control Tags

| Address | Tag | Purpose |
|---|---|---|
| `B3:0/6` | `CONVEYOR_RUN` | Maintained conveyor run state |
| `B3:0/8` | `JAM_FAULT` | Latched jam fault |
| `B3:0/9` | `FAULT_LIGHT` | Internal fault-light command |
| `T4:0` | `PE_BOX_JAM_TIMER` | 5-second photoeye jam timer |

---

## Outputs

| Output | Description |
|---|---|
| `O:0/0` | Conveyor motor command |
| `O:0/3` | Jam fault indicator light |

---

## Control Strategy

### Rung 0000 — Conveyor Start/Stop and Safety Permissives

The conveyor can run only when the E-stop circuit is healthy, the guard is closed, no jam fault is active, and the Stop condition is satisfied.

The Start pushbutton initially energizes `CONVEYOR_RUN`.

A parallel `CONVEYOR_RUN` contact creates a seal-in circuit so the operator does not have to continuously hold the Start button.

If a safety condition or jam fault breaks the rung, `CONVEYOR_RUN` drops out and its seal-in is lost. Restoring the condition therefore does not automatically restart the conveyor.

---

### Rung 0001 — Photoeye Jam Detection

The jam timer runs only when:

`CONVEYOR_RUN = TRUE`

and

`PE_BOX_BLOCK = TRUE`

The timer preset is **5 seconds**.

If the box passes the photoeye before five seconds, the TON resets normally.

If the photoeye remains blocked for the full five seconds, `T4:0/DN` becomes true.

---

### Rung 0002 — Jam Fault Latch

When the jam timer reaches its preset:

`T4:0/DN = TRUE`

an `OTL` instruction latches `JAM_FAULT`.

Using a latched fault allows the timer to reset after the conveyor stops without losing the stored fault condition.

---

### Rung 0003 — Fault Indication

When `JAM_FAULT` is active, `FAULT_LIGHT` is energized.

The fault indication follows the stored fault rather than the timer itself.

---

### Rung 0004 — Jam Fault Reset

The jam fault can only be cleared when:

`RESET_PB = TRUE`

and

`PE_BOX_BLOCK = FALSE`

An `OTU` instruction clears `JAM_FAULT`.

If the obstruction is still blocking the photoeye, the reset is prevented.

---

## Normal Operating Sequence

```text
E-stop healthy
        ↓
Guard closed
        ↓
No active jam fault
        ↓
Operator presses START
        ↓
CONVEYOR_RUN energizes
        ↓
Seal-in maintains conveyor operation
        ↓
Box enters photoeye
        ↓
Jam timer begins
        ↓
Box leaves before 5 seconds
        ↓
Timer resets
        ↓
Conveyor continues running
