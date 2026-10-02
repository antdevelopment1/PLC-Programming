# Generator Auto-Start — Physical Inputs, XIC/XIO & Timer Logic

## Project Overview

This project expanded my generator Start/Stop program by adding an automatic start sequence when utility power is lost.

The biggest challenge was understanding the relationship between:

- The physical pushbutton/contact
- The PLC input bit
- The XIC/XIO instruction in ladder logic

At first, I was trying to read the ladder symbol as if it directly represented how the physical device was wired. That made the Stop pushbutton especially confusing.

## What I Learned

The physical device determines whether the PLC input receives a `0` or `1`.

The ladder instruction then evaluates that value.

The order is:

```text
Physical Device
      ↓
Voltage at PLC Input
      ↓
PLC Input Bit = 0 or 1
      ↓
XIC / XIO evaluates the bit
```
<img width="1215" height="339" alt="Screenshot 2026-10-02 at 2 19 05 AM" src="https://github.com/user-attachments/assets/b4e3c62e-ccf1-40bc-9ce2-d80a6441d8cd" />
<img width="1217" height="661" alt="Screenshot 2026-10-02 at 2 19 12 AM" src="https://github.com/user-attachments/assets/19d228e9-0029-4fba-acbc-dc561f7fdf4e" />
<img width="1205" height="672" alt="Screenshot 2026-10-02 at 2 18 56 AM" src="https://github.com/user-attachments/assets/6df925d7-f6a3-473b-869f-43315f2cf22d" />


### Start vs Stop Pushbuttons

The Start and Stop buttons can both use an XIC even though the physical contacts behave differently.

| Device | Physical Contact | Resting Input | Pressed Input | PLC Logic |
|---|---|---:|---:|---|
| Start | Normally Open | 0 | 1 | XIC |
| Stop | Normally Closed | 1 | 0 | XIC |

For the Start button:

```text
Pressed
→ contact closes
→ input = 1
→ XIC becomes true
→ generator can start
```

For the Stop button:

```text
Not pressed
→ contact is closed
→ input = 1
→ XIC is true

Pressed
→ contact opens
→ input = 0
→ XIC becomes false
→ generator stops
```

This helped me understand that the ladder symbol alone does not tell me how the physical device is wired.

## Automatic Generator Start

I also added a timer that detects a loss of utility power.

```text
Utility power lost
      ↓
TON begins timing
      ↓
3 seconds pass
      ↓
T4:1/DN becomes true
      ↓
Start Generator Relay energizes
```

The `T4:1/DN` contact acts like an **automatic Start command**.

It is not the seal-in.

The three parallel paths are:

```text
Manual Start
     OR
Start Relay Seal-In
     OR
Utility Dead Timer Done
```

The `Start Gen Relay` contact is still responsible for sealing the circuit in after it starts.

## Main Takeaway

My biggest takeaway from this project was:

> Understand the physical device first, determine what PLC input state it produces, and then decide how the ladder logic should evaluate that state.

I also learned that two devices can use the exact same XIC instruction while behaving completely differently because their physical contacts are wired differently.
