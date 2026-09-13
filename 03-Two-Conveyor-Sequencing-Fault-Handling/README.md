# Two-Conveyor Sequencing with Delayed Start/Stop and Fault Handling

A PLC programming exercise built in **RSLogix Micro Starter Lite** to practice coordinated machine sequencing, timing, fault handling, alarms, and one-shot logic.

## Project Objective

Control two conveyors as a coordinated system rather than as independent motors.

The program is designed to demonstrate how a PLC can:

- Start equipment in a defined sequence
- Introduce controlled delays between machine actions
- Stop equipment in the proper order
- Detect fault conditions
- Prevent operation when fault conditions are active
- Generate alarm and notification signals
- Use one-shot logic for events that should occur only once

## Platform

- **Software:** RSLogix Micro Starter Lite
- **PLC:** Allen-Bradley MicroLogix 1100 Series B
- **Language:** Ladder Logic

## Program Structure

The project separates logic into dedicated ladder files:

- `MAIN` – Main program execution
- `CONTROLS` – Operator commands and control logic
- `IO` – Input/output handling
- `ALARMS` – Alarm detection and notification logic

This keeps machine control, physical I/O, and alarm behavior separated instead of placing all logic in one ladder file.

## Concepts Demonstrated

### Conveyor Sequencing

The conveyors operate according to a defined sequence rather than starting simultaneously.

Timing logic controls when the next conveyor is allowed to start or stop.

### Delayed Start / Stop

Timers are used to create controlled delays between conveyor actions.

This introduces the concept of coordinating multiple pieces of equipment through PLC sequencing.

### Fault Handling

Fault conditions can interrupt normal machine operation and prevent the sequence from continuing until the required conditions are restored.

### Alarm Logic

Alarm conditions are separated from machine control logic so that faults can be detected, stored, and communicated independently.

### One-Shot Logic

One-shot instructions are used when an event should occur for only **one PLC scan** when a condition first becomes true.

This prevents an action from repeatedly executing during every scan while the triggering condition remains active.

### Storage vs. Output Bits

The project also separates internal logic states from output/notification states.

This allows the PLC to remember that an event occurred while controlling separately how that event is communicated elsewhere in the program.

## Repository Contents

```text
03-Two-Conveyor-Sequencing-Fault-Handling/
├── README.md
├── two-conveyor-sequencing-fault-handling.rss
└── screenshots/
```

## Skills Practiced

- Ladder logic
- PLC scan-cycle thinking
- Timers
- Internal binary bits
- Sequencing
- Interlocks
- Fault handling
- Alarm logic
- One-shot instructions
- Program organization
- Troubleshooting logic across multiple ladder files

## Purpose

This project is part of a larger controls-engineering practice portfolio focused on progressing from individual PLC instructions to complete machine-control patterns.

The goal is not simply to make the ladder logic run, but to understand **why each instruction exists, what state it represents, and how that state is used elsewhere in the program**.

## Screenshots
<img width="1427" height="630" alt="Screenshot 2026-09-12 at 8 52 01 PM" src="https://github.com/user-attachments/assets/e39b6dad-2735-4f3f-9575-683e7eb182b9" />
<img width="1440" height="322" alt="Screenshot 2026-09-12 at 8 51 50 PM" src="https://github.com/user-attachments/assets/328b9bc8-b8fd-4876-870f-c18bd8a33868" />
<img width="1440" height="657" alt="Screenshot 2026-09-12 at 8 51 44 PM" src="https://github.com/user-attachments/assets/6dc30137-c2ab-4f05-ba3d-31825c2ba0c3" />
