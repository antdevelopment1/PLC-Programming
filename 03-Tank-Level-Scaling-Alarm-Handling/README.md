# Tank Level Scaling, Alarm & Fault Handling

## 🔍 Overview

A PLC process-control project that monitors tank level through an analog input, scales the raw signal into engineering units, controls pump and valve outputs, and detects abnormal Low-Low and High-High level conditions.

The project demonstrates analog scaling, compare instructions, delayed alarm qualification, hysteresis/reset thresholds, one-shot event detection, process alarms, and separate operator notifications.

## ⚙️ Platform & Tools

* **Software:** RSLogix Micro Starter Lite
* **PLC Family:** Allen-Bradley MicroLogix
* **Language:** Ladder Logic
* **Analog Scaling:** SCP — Scale with Parameters
* **Timers:** TON
* **Event Detection:** One-Shot Rising
* **Comparison Instructions:** LES / GRT

## 🗺️ System I/O & Tags

### Analog Input

* `I:3.0` - **Tank Level Raw Input** - Raw analog signal representing tank level.

### Physical Outputs

* `O:2/0` - **PUMP** - Physical output controlled by the internal pump command.
* `O:2/1` - **VALVE** - Physical output controlled by the internal valve command.

### Internal Control / Process Data

* `B3:0/4` - **PUMP** - Internal pump command.
* `B3:0/5` - **VALVE** - Internal valve command.
* `N7:0` - **LEVEL** - Scaled tank-level value represented from 0–100%.

### Alarm Bits

* `B3:0/0` - **LL_ALARM** - Low-Low process alarm.
* `B3:0/1` - **LL_NOTIFICATION** - Operator notification for a Low-Low event.
* `B3:0/2` - **ALARM_RESET** - Operator acknowledgement/reset command.
* `B3:0/11` - **HH_ALARM** - High-High process alarm.
* `B3:0/12` - **HH_NOTIFICATION** - Operator notification for a High-High event.

### Timers

* `T4:2` - **Low-Low Timer** - Requires the tank level to remain below 15% for 5 seconds before generating a Low-Low event.
* `T4:3` - **High-High Timer** - Requires the tank level to remain above 85% for 5 seconds before generating a High-High event.

## 📐 Analog Level Scaling

The physical analog level signal enters the PLC through:

`I:3.0`

An `SCP` instruction converts the raw analog signal into a 0–100% tank-level value.

| Parameter | Value |
| --- | ---: |
| Input Minimum | 0 |
| Input Maximum | 16383 |
| Scaled Minimum | 0 |
| Scaled Maximum | 100 |
| Output | `N7:0 LEVEL` |

Conceptually:

**Raw Analog Input → SCP Scaling → 0–100% Tank Level**

This allows the control and alarm logic to work with meaningful engineering units instead of raw analog counts.

## 🛠️ Program Structure

The program separates control, I/O, and alarm behavior into dedicated ladder files.

### MAIN Routine

The Main routine executes three subroutines:

* **Rung 0000:** `CONTROLS` (`U:3`)
* **Rung 0001:** `DIGITAL IO` (`U:4`)
* **Rung 0002:** `ALARMS` (`U:5`)

This keeps process control, hardware interfacing, and alarm handling separated into distinct program areas.

### DIGITAL IO Routine

The I/O routine connects internal PLC commands to physical outputs:

* `B3:0/4 PUMP` → `O:2/0 PUMP`
* `B3:0/5 VALVE` → `O:2/1 VALVE`

The same routine scales the raw analog tank-level input:

`I:3.0 → SCP → N7:0 LEVEL`

## 🚨 Alarm Strategy

The alarm logic separates three concepts:

**Process Condition → Alarm → Operator Notification**

A brief threshold crossing does not immediately generate an alarm. The abnormal level must remain present for five seconds before the event is recognized.

The process alarm and operator notification are also handled differently:

* The **alarm** remains active until the tank level recovers to a defined reset threshold.
* The **notification** remains active until the operator acknowledges the event.

This creates a more deliberate alarm-handling strategy and prevents brief process fluctuations from generating unnecessary alarms.

## ⬇️ Low-Low Level Alarm

### Alarm Trigger

The Low-Low sequence begins when:

`LEVEL < 15%`

If the tank remains below 15% continuously for five seconds:

`T4:2/DN = TRUE`

The completed timer generates the Low-Low event.

### Low-Low Alarm

The event activates:

`B3:0/0 LL_ALARM`

The alarm does not immediately clear when the level rises back above 15%.

Instead, it remains active until the tank recovers to approximately:

`LEVEL >= 20%`

This creates a **5% hysteresis band** between the alarm trigger and recovery point.

### Low-Low Notification

The same Low-Low event also generates:

`B3:0/1 LL_NOTIFICATION`

Unlike the process alarm, the notification remains active until the operator acknowledges the event through:

`B3:0/2 ALARM_RESET`

This allows the process condition to recover independently while still requiring operator awareness of the abnormal event.

## ⬆️ High-High Level Alarm

### Alarm Trigger

The High-High sequence begins when:

`LEVEL > 85%`

If the tank remains above 85% continuously for five seconds:

`T4:3/DN = TRUE`

The completed timer generates the High-High event.

### High-High Alarm

The event activates:

`B3:0/11 HH_ALARM`

The alarm remains active after the level falls below the original 85% trigger point.

It clears only after the tank level falls to approximately:

`LEVEL <= 80%`

This creates a **5% hysteresis band** between the High-High alarm trigger and its recovery point.

### High-High Notification

The High-High event also generates:

`B3:0/12 HH_NOTIFICATION`

The notification remains active until the operator acknowledges the event using:

`B3:0/2 ALARM_RESET`

## 🔄 Alarm Behavior Summary

| Condition | Qualification | Alarm Recovery |
| --- | --- | --- |
| Low-Low | `LEVEL < 15%` for 5 seconds | Level recovers to 20% |
| High-High | `LEVEL > 85%` for 5 seconds | Level falls to 80% |

Both events also create separate operator notifications that remain active until acknowledged.

## 🧠 Key Control Concepts Demonstrated

* Analog input scaling
* Engineering-unit conversion
* LES and GRT compare instructions
* TON delayed alarm qualification
* One-shot event detection
* Low-Low and High-High process alarms
* Alarm hysteresis
* Separate alarm and notification behavior
* Operator acknowledgement
* Separation of Controls, I/O, and Alarm routines
* Internal control commands mapped to physical outputs

## 🛡️ Alarm & Recovery Philosophy

The project distinguishes between an active process condition and an operator notification.

A threshold violation must persist for five seconds before becoming an alarm. Once an alarm is active, the process must recover beyond a separate reset threshold before the alarm clears.

The notification remains active independently so that an abnormal event cannot occur and disappear without operator awareness.

> **Safety Note:** This is a PLC training project demonstrating process-control and alarm-management concepts. The logic shown is not a substitute for safety-rated level protection, independent overfill protection, or other required process-safety systems.
