# Conveyor Control with Jam Detection and Fault Recovery

## 🔍 Overview

A PLC conveyor-control project that combines start/stop control, operating permissives, photoeye-based jam detection, fault latching, operator indication, and controlled fault recovery.

The conveyor runs only when its required permissives are satisfied. While the conveyor is running, a blocked photoeye starts a five-second timer. If the obstruction remains long enough for the timer to complete, the PLC latches a jam fault, stops the conveyor, activates fault indication, and prevents restart until the obstruction has been cleared and the operator performs a reset.

## ⚙️ Platform & Tools

* **Software:** RSLogix Micro Starter Lite
* **PLC Family:** Allen-Bradley MicroLogix
* **Language:** Ladder Logic
* **Timer Instruction:** TON
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

* `O:0/0` - **CONVEYOR MOTOR** - Physical output controlled by the internal conveyor run command.
* `O:0/1` - **RUN COMMAND** - Output mapped from `B3:0/7 RUN_COMMAND`.
* `O:0/2` - **JAM FAULT** - Output reflecting the latched jam-fault state.
* `O:0/3` - **FAULT LIGHT** - Physical fault indication output.
* `O:0/4` - **JAM TIMER** - Output mapped from `B3:0/10 JAM_TIMER`.

> `RUN_COMMAND` and `JAM_TIMER` are present in the Digital I/O mapping, but the control logic shown in this project does not currently drive `B3:0/7` or `B3:0/10`. The actual jam timing function is performed by `T4:0`.

### Internal Control Bits / Timers

* `B3:0/0` - **START_PB** - Internal mapped state of the Start pushbutton.
* `B3:0/1` - **STOP_PB** - Internal mapped state of the Stop pushbutton.
* `B3:0/2` - **RESET_PB** - Internal mapped state of the Reset pushbutton.
* `B3:0/3` - **ESTOP_OK** - Internal mapped E-stop permissive.
* `B3:0/4` - **GUARD_OK** - Internal mapped guard permissive.
* `B3:0/5` - **PE_BOX_BLOCK** - Internal mapped photoeye state.
* `B3:0/6` - **CONVEYOR_RUN** - Internal conveyor run command and seal-in bit.
* `B3:0/7` - **RUN_COMMAND** - Internal bit mapped to `O:0/1`; no driving control rung is shown in the current logic.
* `B3:0/8` - **JAM_FAULT** - Latched jam-fault memory bit.
* `B3:0/9` - **FAULT_LIGHT** - Internal fault-indication command.
* `B3:0/10` - **JAM_TIMER** - Internal bit mapped to `O:0/4`; separate from the actual `T4:0` timer.
* `T4:0` - **PE_BOX_JAM_TIMER** - TON used to detect a persistent photoeye blockage.
  * **Time Base:** 1.0 second
  * **Preset:** 5 seconds

## 🛠️ Control Strategy & Key Rungs

The program separates physical I/O from machine-control decisions:

**Physical Inputs → Internal B3 Bits → Control Logic → Internal Commands → Physical Outputs**

### MAIN Routine

The Main routine coordinates program execution through two subroutines:

* **Rung 0000:** Calls `DIGITAL IO` (`U:3`)
* **Rung 0001:** Calls `CONTROLS` (`U:4`)

This keeps field I/O mapping separate from the conveyor's operating and fault logic.

### DIGITAL IO Routine

The Digital I/O routine maps physical PLC inputs into internal memory:

* `I:0/0` → `B3:0/0 START_PB`
* `I:0/1` → `B3:0/1 STOP_PB`
* `I:0/2` → `B3:0/2 RESET_PB`
* `I:0/3` → `B3:0/3 ESTOP_OK`
* `I:0/4` → `B3:0/4 GUARD_OK`
* `I:0/5` → `B3:0/5 PE_BOX_BLOCK`

It also maps internal commands/status bits to physical outputs:

* `B3:0/6 CONVEYOR_RUN` → `O:0/0 CONVEYOR MOTOR`
* `B3:0/7 RUN_COMMAND` → `O:0/1 RUN COMMAND`
* `B3:0/8 JAM_FAULT` → `O:0/2 JAM FAULT`
* `B3:0/9 FAULT_LIGHT` → `O:0/3 FAULT LIGHT`
* `B3:0/10 JAM_TIMER` → `O:0/4 JAM TIMER`

## 🛠️ Controls Routine

### Rung 0000 — Conveyor Start/Stop and Permissives

The conveyor can run only when:

* `ESTOP_OK` is true
* The Stop command is not active
* `GUARD_OK` is true
* `JAM_FAULT` is not active
* Start has been requested or the conveyor is already sealed in

`CONVEYOR_RUN` is placed in parallel with the momentary Start command to create the seal-in circuit.

This allows the conveyor to continue running after the Start pushbutton is released while immediately removing the run command if a required permissive is lost.

### Rung 0001 — Photoeye Jam Detection Timer

The jam timer runs only when:

* `CONVEYOR_RUN` is active, and
* `PE_BOX_BLOCK` is active.

These conditions start `T4:0 PE_BOX_JAM_TIMER`.

The timer has:

* **Time base:** 1 second
* **Preset:** 5 seconds

A brief photoeye interruption therefore does not create a jam fault. The obstruction must remain continuously present for five seconds.

### Rung 0002 — Jam Fault Latch

When `T4:0/DN` becomes true, the PLC executes an `OTL` instruction on:

`B3:0/8 JAM_FAULT`

The fault therefore remains stored even after the conveyor stops and the timer subsequently resets.

Because `JAM_FAULT` is also used as an XIO interlock in Rung 0000, the newly latched fault breaks the conveyor seal-in logic and removes `CONVEYOR_RUN`.

### Rung 0003 — Jam Fault Indication

When `B3:0/8 JAM_FAULT` is active, the PLC energizes:

`B3:0/9 FAULT_LIGHT`

The Digital I/O routine then maps this command to:

`O:0/3 FAULT LIGHT`

This provides a visible indication that the machine is in a faulted condition.

### Rung 0004 — Jam Fault Reset

The jam fault can be cleared only when:

* `RESET_PB` is pressed, and
* `PE_BOX_BLOCK` is false.

An `OTU` instruction then unlatches:

`B3:0/8 JAM_FAULT`

Requiring the photoeye to be clear prevents the operator from resetting the fault while the condition that caused the jam is still present.

## 🛡️ Safety & Fault Recovery Behavior

### Operating Permissives

The conveyor run command depends on:

* E-stop permissive healthy
* Guard permissive healthy
* Stop command inactive
* No active jam fault

Loss of a required permissive breaks the run circuit and causes `CONVEYOR_RUN` to de-energize.

### Jam Detection

A blocked photoeye does **not** immediately fault the machine.

The PLC requires:

**Conveyor Running + Photoeye Blocked continuously for 5 seconds**

before declaring a jam.

This prevents normal short-duration product detection from being interpreted as a fault.

### Fault Response

Once the five-second timer reaches done:

1. `JAM_FAULT` is latched.
2. The active jam-fault interlock breaks the conveyor run circuit.
3. `CONVEYOR_RUN` drops out.
4. The conveyor motor output turns off.
5. The fault-light command energizes.
6. The fault remains stored until a valid reset occurs.

### Recovery Sequence

To recover from a jam:

1. Identify and remove the obstruction.
2. Confirm the photoeye is clear.
3. Confirm the E-stop and guard permissives are healthy.
4. Press the Reset pushbutton.
5. The PLC unlatches `JAM_FAULT`.
6. The fault indication clears.
7. Press Start to begin a new conveyor run.

The conveyor does **not automatically restart** when the fault is reset because the original `CONVEYOR_RUN` seal-in was broken when the fault occurred. A new Start command is required.

> **Safety Note:** `ESTOP_OK` and `GUARD_OK` are used as standard PLC permissives in this training project. The logic shown is not a safety-rated implementation of an emergency-stop or machine-guarding system. Personnel-protection functions require appropriate safety-rated hardware and circuit design.


<img width="1436" height="713" alt="Screenshot 2026-09-13 at 10 25 04 PM" src="https://github.com/user-attachments/assets/7bf88c0f-a699-4185-941f-4ba9926ed062" />
<img width="1433" height="694" alt="Screenshot 2026-09-13 at 10 25 13 PM" src="https://github.com/user-attachments/assets/0f023050-7f88-4833-a602-6a33b903361a" />
<img width="1218" height="347" alt="Screenshot 2026-09-13 at 10 25 34 PM" src="https://github.com/user-attachments/assets/3705261f-30ba-447e-9a01-19700a62b7f3" />
<img width="1213" height="441" alt="Screenshot 2026-09-13 at 10 25 49 PM" src="https://github.com/user-attachments/assets/f2efc1fe-7d21-453e-a377-b70b58e46da0" />
<img width="1209" height="536" alt="Screenshot 2026-09-13 at 10 25 56 PM" src="https://github.com/user-attachments/assets/68df7623-270a-417b-85d3-507e26f2a0de" />
