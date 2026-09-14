# Hand/Off/Auto Blower Control

## 🔍 Overview

A foundational PLC control project that implements Hand/Off/Auto operating modes for an industrial blower. The project separates physical operator inputs, internal machine-state logic, and the physical blower output while using one-shot instructions and an integer state value to control operating-mode transitions.

The program demonstrates a basic machine-control architecture in which field inputs are mapped into internal PLC memory, control logic determines the desired machine state, and a separate I/O routine maps the resulting command to the physical output.

## ⚙️ Platform & Tools

* **Software:** RSLogix Micro Starter Lite
* **PLC:** Allen-Bradley MicroLogix 1100
* **Language:** Ladder Logic

## 🗺️ System I/O & Tags

### Physical Inputs

* `I:1/0` - **HAND PB INPUT** - Operator pushbutton requesting Hand mode.
* `I:1/1` - **OFF PB INPUT** - Operator pushbutton requesting Off mode.
* `I:1/2` - **AUTO PB INPUT** - Operator pushbutton requesting Auto mode.

### Physical Outputs

* `O:2/0` - **BLOWER OUTPUT** - Physical PLC output controlled by the internal blower command.

### Internal Control Bits / Data

* `B3:0/0` - **HAND_PB** - Internal mapped state of the Hand pushbutton.
* `B3:0/1` - **OFF_PB** - Internal mapped state of the Off pushbutton.
* `B3:0/2` - **AUTO_PB** - Internal mapped state of the Auto pushbutton.
* `B3:0/3` - **BLOWER** - Internal blower run command mapped to the physical blower output.
* `B3:0/4` - **HAND ONE-SHOT** - ONS storage bit used when processing the Hand command.
* `B3:0/5` - **OFF ONE-SHOT** - ONS storage bit used when processing the Off command.
* `B3:0/6` - **AUTO ONE-SHOT** - ONS storage bit used when processing the Auto command.
* `B3:0/7` - **BLOWER AUTO ENERGIZE BIT** - Automatic run-enable condition used while the blower is in Auto mode.
* `N7:0` - **BLOWER STATE** - Stores the current operating mode:
  * `0` = Off
  * `1` = Hand
  * `2` = Auto

No timers are used in this project.

## 🛠️ Control Strategy & Key Rungs

The program follows a separation-of-responsibility pattern:

**Physical Input → Internal Input Bit → Mode/State Logic → Internal Run Command → Physical Output**

### MAIN Routine

The `MAIN` routine calls the program's two functional subroutines:

* **Rung 0000:** Calls `DIGITAL IO` (`U:3`)
* **Rung 0001:** Calls `CONTROLS` (`U:4`)

This keeps physical I/O handling separate from the machine's operating logic.

### DIGITAL IO Routine

The Digital I/O routine maps physical field signals into internal PLC memory and maps the internal blower command back to the physical output.

* **Rung 0000:** `I:1/0` HAND PB INPUT → `B3:0/0` HAND_PB
* **Rung 0001:** `I:1/1` OFF PB INPUT → `B3:0/1` OFF_PB
* **Rung 0002:** `I:1/2` AUTO PB INPUT → `B3:0/2` AUTO_PB
* **Rung 0003:** `B3:0/3` BLOWER → `O:2/0` BLOWER OUTPUT

### CONTROLS Routine

The Controls routine stores the selected operating mode in `N7:0` rather than controlling the physical output directly from the pushbuttons.

* **Rung 0000 — Hand Selection:**  
  When the Hand pushbutton is pressed, an ONS allows the command to execute once. If the current blower state is not Auto, a `MOV` writes `1` into `N7:0`, placing the blower in Hand mode.

* **Rung 0001 — Off Selection:**  
  When the Off pushbutton is pressed, an ONS triggers a `MOV` instruction that writes `0` into `N7:0`.

* **Rung 0002 — Auto Selection:**  
  When the Auto pushbutton is pressed, an ONS triggers a `MOV` instruction that writes `2` into `N7:0`.

* **Rung 0003 — Blower Run Decision:**  
  The internal `B3:0/3` BLOWER command energizes under either of two conditions:

  1. `N7:0 = 1` — the blower is in **Hand** mode, or
  2. `N7:0 = 2` — the blower is in **Auto** mode **and** `B3:0/7` BLOWER AUTO ENERGIZE BIT is true.

This allows Hand operation to command the blower directly while Auto mode requires a separate automatic run condition.

## 🛡️ Safety & Fault Recovery Behavior

This introductory project focuses on operating modes, state memory, one-shot logic, and separation of I/O from control logic. Dedicated equipment-fault detection and safety-rated control are not implemented.

* **Mode Interlock:** A `NEQ` instruction prevents a direct transition from Auto (`N7:0 = 2`) into Hand mode. The operator must first leave Auto before Hand can be selected.
* **Off Behavior:** Selecting Off writes `0` to the blower state, causing the blower run command to de-energize.
* **Auto Behavior:** Selecting Auto stores the Auto state, but the blower only runs while `B3:0/7` BLOWER AUTO ENERGIZE BIT is true.
* **Auto Demand Loss:** If the automatic enable bit becomes false, the blower command turns off while the controller remains in Auto mode. If the automatic enable returns, the blower can be commanded on again.
* **Fault Handling:** No overload, failed-to-start, E-stop, or other equipment-fault logic is implemented in this project.
* **Fault Recovery:** No dedicated fault-reset or recovery sequence is implemented.

> **Safety Note:** This project demonstrates standard PLC operating-mode and control-logic concepts for training purposes. It does not implement safety-rated functions such as emergency-stop or personnel-protection circuits.


<img width="1202" height="291" alt="main" src="https://github.com/user-attachments/assets/a1dbd8e8-8034-4c7e-a0c0-3d5132d2e4f9" />
<img width="1200" height="409" alt="digital io" src="https://github.com/user-attachments/assets/5821a2c8-3d5c-4cc7-9bd1-d5275e979d42" />
<img width="1202" height="639" alt="controls" src="https://github.com/user-attachments/assets/34549509-f103-4abb-9cee-1f6d8881710b" />
