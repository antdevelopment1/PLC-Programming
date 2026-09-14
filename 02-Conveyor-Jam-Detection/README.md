# Conveyor Control with Jam Detection and Fault Recovery

## 🔍 Overview

A PLC conveyor-control project that combines start/stop control, operating permissives, photoeye-based jam detection, fault latching, operator indication, and controlled fault recovery.

The conveyor runs only when its required permissives are satisfied. While the conveyor is running, a blocked photoeye starts a five-second timer. If the obstruction remains long enough for the timer to complete, the PLC latches a jam fault, stops the conveyor, activates fault indication, and prevents restart until the obstruction has been cleared and the operator performs a reset.

## ⚙️ Platform & Tools

* **Software:** RSLogix Micro Starter Lite
* **PLC:** Allen-Bradley MicroLogix 1100 / 1763 family
* **Output Module:** 1762-OW8
* **Language:** Ladder Logic
* **Timer:** TON
* **Fault Memory:** OTL / OTU latch-unlatch logic

## 🗺️ System I/O & Tags

### Physical Inputs

* `I:0/0` - **START PB** - Momentary pushbutton requesting conveyor startup.
* `I:0/1` - **STOP PB** - Operator stop command.
* `I:0/2` - **RESET PB** - Operator command used to reset a cleared jam fault.
* `I:0/3` - **ESTOP OK** - Indicates that the E-stop permissive is healthy.
* `I:0/4` - **GUARD OK** - Indicates that the guard permissive is healthy.
* `I:0/5` - **PE BOX BLOCK** - Photoeye signal indicating that a box or obstruction is blocking the sensor.

### Physical Outputs

* `O:2/0` - **CONVEYOR RUN** - Physical output controlled by the internal conveyor run command.
* `O:2/1` - **JAM FAULT** - Physical output indicating that a jam fault is latched.
* `O:2/2` - **FAULT LIGHT** - Physical output used for visible fault indication.

### Internal Control Bits / Timers

* `B3:0/0` - **START_PB** - Internal mapped state of the Start pushbutton.
* `B3:0/1` - **STOP_PB** - Internal mapped state of the Stop pushbutton.
* `B3:0/2` - **RESET_PB** - Internal mapped state of the Reset pushbutton.
* `B3:0/3` - **ESTOP_OK** - Internal mapped E-stop permissive.
* `B3:0/4` - **GUARD_OK** - Internal mapped guard permissive.
* `B3:0/5` - **PE_BOX_BLOCK** - Internal mapped photoeye state.
* `B3:0/6` - **CONVEYOR_RUN** - Internal conveyor run command and seal-in bit.
* `B3:0/8` - **JAM_FAULT** - Latched jam-fault memory bit.
* `B3:0/9` - **FAULT_LIGHT** - Internal fault-light command.
* `T4:0` - **PE_BOX_JAM_TIMER** - TON used to detect a persistent photoeye blockage.
  * **Time Base:** 1.0 second
  * **Preset:** 5 seconds

## 🛠️ Control Strategy & Key Rungs

The program separates physical I/O from machine-control decisions:

**Physical Inputs → Internal B3 Bits → Control Logic → Internal Commands → Physical Outputs**

### MAIN Routine

The Main routine coordinates execution through two subroutines:

* **Rung 0000:** Calls `DIGITAL IO` (`U:3`)
* **Rung 0001:** Calls `CONTROLS` (`U:4`)

This keeps physical I/O mapping separate from the conveyor's control and fault logic.

### DIGITAL IO Routine

The Digital I/O routine maps physical field inputs into internal PLC memory:

* `I:0/0` → `B3:0/0 START_PB`
* `I:0/1` → `B3:0/1 STOP_PB`
* `I:0/2` → `B3:0/2 RESET_PB`
* `I:0/3` → `B3:0/3 ESTOP_OK`
* `I:0/4` → `B3:0/4 GUARD_OK`
* `I:0/5` → `B3:0/5 PE_BOX_BLOCK`

Internal control and fault bits are then mapped to physical outputs:

* `B3:0/6 CONVEYOR_RUN` → `O:2/0 CONVEYOR RUN`
* `B3:0/8 JAM_FAULT` → `O:2/1 JAM FAULT`
* `B3:0/9 FAULT_LIGHT` → `O:2/2 FAULT LIGHT`

## ⚙️ CONTROLS Routine

### Rung 0000 — Conveyor Start/Stop and Safety Permissives

The conveyor can run only when:

* `ESTOP_OK` is true
* The Stop command is inactive
* `GUARD_OK` is true
* `JAM_FAULT` is not active
* Start is pressed or `CONVEYOR_RUN` is already sealed in

`CONVEYOR_RUN` is placed in parallel with the momentary Start command to form the seal-in circuit.

Once started, the conveyor continues running after the Start pushbutton is released. Loss of a required permissive breaks the circuit and removes the conveyor run command.

### Rung 0001 — Photoeye Jam Detection Timer

Jam timing occurs only when:

* `CONVEYOR_RUN` is active, and
* `PE_BOX_BLOCK` is active.

These conditions enable:

`T4:0 PE_BOX_JAM_TIMER`

The timer uses a **1.0-second time base** and a **5-second preset**.

This means a normal short-duration photoeye blockage does not immediately generate a fault. The photoeye must remain continuously blocked for five seconds.

### Rung 0002 — Jam Fault Latch

When:

`T4:0/DN = TRUE`

the PLC executes an `OTL` instruction on:

`B3:0/8 JAM_FAULT`

The fault remains stored even after the timer condition disappears.

Because `JAM_FAULT` is also used as an XIO interlock in the conveyor run rung, the newly latched fault breaks the conveyor seal-in circuit and causes `CONVEYOR_RUN` to de-energize.

### Rung 0003 — Jam Fault Indication

When:

`B3:0/8 JAM_FAULT`

is active, the PLC energizes:

`B3:0/9 FAULT_LIGHT`

The Digital I/O routine then maps:

`B3:0/9 FAULT_LIGHT → O:2/2 FAULT LIGHT`

A separate physical jam-fault output is also driven directly from the latched fault:

`B3:0/8 JAM_FAULT → O:2/1 JAM FAULT`

### Rung 0004 — Jam Fault Reset

The jam fault can only be cleared when:

* `RESET_PB` is pressed, and
* `PE_BOX_BLOCK` is false.

The PLC then executes an `OTU` instruction on:

`B3:0/8 JAM_FAULT`

Requiring the photoeye to be clear prevents the fault from being reset while the obstruction that caused it is still present.

## 🛡️ Safety & Fault Recovery Behavior

### Operating Permissives

The conveyor requires:

* E-stop permissive healthy
* Guard permissive healthy
* Stop command inactive
* No active jam fault

Loss of any required permissive causes the conveyor run command to drop out.

### Jam Detection

A blocked photoeye does not immediately fault the conveyor.

The fault condition requires:

**Conveyor Running + Photoeye Blocked Continuously for 5 Seconds**

Only after the five-second timer reaches done is the jam fault latched.

### Fault Response

When a jam is detected:

1. `T4:0/DN` becomes true.
2. `JAM_FAULT` is latched.
3. The jam-fault interlock breaks the conveyor run circuit.
4. `CONVEYOR_RUN` drops out.
5. `O:2/0 CONVEYOR RUN` turns off.
6. `O:2/1 JAM FAULT` turns on.
7. `FAULT_LIGHT` energizes.
8. `O:2/2 FAULT LIGHT` turns on.
9. The jam fault remains latched until a valid reset occurs.

### Recovery Sequence

To recover from a jam:

1. Identify and remove the obstruction.
2. Confirm that the photoeye is clear.
3. Confirm that the E-stop and guard permissives are healthy.
4. Press the Reset pushbutton.
5. The PLC unlatches `JAM_FAULT`.
6. The jam-fault and fault-light outputs clear.
7. Press Start to begin a new conveyor run.

The conveyor does **not automatically restart** after the fault is reset. Because the original `CONVEYOR_RUN` seal-in was broken by the fault, a new Start command is required.

> **Safety Note:** `ESTOP_OK` and `GUARD_OK` are represented as standard PLC permissives for this training exercise. This program does not represent a safety-rated emergency-stop or machine-guarding implementation. Personnel-protection functions require appropriate safety-rated hardware and circuit design.


<img width="1182" height="657" alt="Screenshot 2026-09-13 at 10 39 50 PM" src="https://github.com/user-attachments/assets/2d97f477-d5a1-42c1-9870-1a6d9dcd92b3" />
<img width="1192" height="676" alt="Screenshot 2026-09-13 at 10 39 57 PM" src="https://github.com/user-attachments/assets/a2528f96-4173-452a-b30d-c489ac013382" />
M" src="https://github.com/user-attachments/assets/68df7623-270a-417b-85d3-507e26f2a0de" />
<img width="1214" height="311" alt="Screenshot 2026-09-13 at 10 41 46 PM" src="https://github.com/user-attachments/assets/7e13ca97-12ac-46b8-9472-9900ebb0e3db" />
<img width="1170" height="458" alt="Screenshot 2026-09-13 at 10 41 58 PM" src="https://github.com/user-attachments/assets/42c7e9bc-e0fc-448d-bff4-9e572e411d02" />
<img width="1189" height="478" alt="Screenshot 2026-09-13 at 10 42 19 PM" src="https://github.com/user-attachments/assets/403e87ad-71d8-4505-bd47-0260712c8c7a" />

