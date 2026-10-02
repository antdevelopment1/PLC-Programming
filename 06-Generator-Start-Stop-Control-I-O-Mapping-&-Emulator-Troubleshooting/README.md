```markdown
# Generator Start/Stop Control — I/O Mapping, Signal Flow & Emulator Troubleshooting

## Project Overview

This project started as a basic generator Start/Stop control exercise, but it became a much deeper lesson in how data actually moves through a PLC program.

The biggest things I learned were:

- How an I/O mapping layer works
- Why changing an internal `B3` bit does not always change the system
- How to trace a signal back to its root value
- Why I need to manipulate the actual input or output when testing the complete chain
- The difference between being **ONLINE** with a PLC and the PLC actually being in **RUN**
- How to troubleshoot by finding the exact point where a signal stops changing

---

## Program Structure

Instead of using the physical I/O addresses directly throughout the control logic, I separated the program into layers.

The Main Program calls both routines using JSR instructions:

    MAIN PROGRAM
        ↓
    JSR DIGITAL_IO
        ↓
    JSR CONTROLS

The overall signal path is:

    Physical Input
          ↓
    DIGITAL_IO
          ↓
    Internal B3 Input Bit
          ↓
    CONTROLS
          ↓
    Internal B3 Command Bit
          ↓
    DIGITAL_IO
          ↓
    Physical Output

In simple address form:

    I: → B3 → CONTROL LOGIC → B3 → O:

---

## I/O

### Inputs

| Address | Description |
|---|---|
| `I:0/0` | Start Push Button |
| `I:0/1` | Stop Push Button |
| `I:0/2` | Engine Running |

### Outputs

| Address | Description |
|---|---|
| `O:0/0` | Start Generator Relay |
| `O:0/1` | Engine Running Light |
| `O:0/2` | Engine Stopped Light |

### Internal Bits

| Address | Description |
|---|---|
| `B3:0/0` | Start Pushbutton |
| `B3:0/1` | Stop Push Button |
| `B3:0/2` | Start Generator Relay Command |
| `B3:0/3` | Engine Running |
| `B3:0/4` | Engine Stopped Light Command |
| `B3:0/5` | Engine Running Light Command |

---

## My First Disconnect: Why Changing the B3 Bit Did Not Work

While testing the program, I originally tried changing some of the internal `B3` bits directly.

For example, my input mapping follows this pattern:

    I:0/0 → B3:0/0

I would try changing:

    B3:0/0

but it would not behave the way I expected.

The reason became clear once I understood the order of operations.

The PLC continuously executes the mapping rung:

    I:0/0 -------- OTE B3:0/0

That means `B3:0/0` is not the root source of the Start Pushbutton state.

`I:0/0` is.

If:

    I:0/0 = 0

and I manually change:

    B3:0/0 = 1

the next PLC scan executes the mapping rung again and writes the input condition back into the B3 bit:

    I:0/0 = 0
          ↓
    B3:0/0 = 0

I was trying to change a value downstream instead of changing the value that actually controlled it.

---

## Discovering the Root Value

This led to one of the most important lessons from this project:

> When a value keeps changing back, determine what rung owns that value and trace the signal upstream.

For the Start Pushbutton, the chain is:

    I:0/0
      ↓
    B3:0/0
      ↓
    CONTROLS
      ↓
    B3:0/2
      ↓
    O:0/0

If I want to simulate somebody pressing the Start button, I need to manipulate the actual input:

    I:0/0

instead of only manipulating:

    B3:0/0

Then I can watch the entire chain happen naturally:

    I:0/0 turns ON
          ↓
    B3:0/0 turns ON
          ↓
    Control logic evaluates
          ↓
    B3:0/2 turns ON
          ↓
    O:0/0 turns ON

This helped me stop thinking about the PLC as a collection of unrelated bits and start thinking about it as a chain of cause and effect.

---

## Testing Inputs and Outputs

I also learned that I need to be intentional about **where** I manipulate a value.

### Testing an Input

If I want to simulate something happening in the field, I should manipulate the actual input:

    I:0/x

Then watch the signal move through the program:

    I:0/x
      ↓
    B3 Mapped Input
      ↓
    Control Logic
      ↓
    B3 Command
      ↓
    O:0/x

This allows me to test the complete chain instead of bypassing part of the program.

### Testing an Output

If I specifically want to test the final PLC output independently, I can manipulate or force:

    O:0/x

That answers a different question.

Forcing the output can tell me:

> Can this output point be turned on?

But it does not prove:

> Does my control logic correctly cause this output to turn on?

For an end-to-end test, I need to start with the root input and allow the program to produce the output naturally.

---

## Understanding Order of Operations

My thinking changed from:

    "I need this bit ON, so I'll turn this bit ON."

to:

    "What controls this bit?"
              ↓
    "What controls that value?"
              ↓
    "Where does this signal actually begin?"

For inputs:

    FIELD
      ↓
    I:
      ↓
    B3
      ↓
    LOGIC

For outputs:

    LOGIC
      ↓
    B3
      ↓
    O:
      ↓
    FIELD

This means I can troubleshoot by finding the exact point where the expected state stops changing.

---

## My Second Disconnect: I Was Online, but the Program Still Did Not Work

Later, I restarted the Windows VM while setting up Studio 5000.

When I returned to RSLogix 500, I reconnected to the emulator.

RSLinx could see the emulator.

RSLogix could go online.

My JSR instructions were still present.

My DIGITAL_IO routine was still present.

My CONTROLS routine was still present.

Everything appeared to be connected correctly.

But the program was not behaving correctly.

I could change:

    I:0/0

but the mapped value:

    B3:0/0

was not changing.

At first, this made me think something was wrong with my I/O mapping or JSR structure.

There wasn't.

---

## The Actual Problem: The Emulator Was Not in RUN

The emulated processor was connected, but it was not in **RUN mode**.

That taught me an important distinction:

    ONLINE ≠ RUNNING

Being online means:

    "My computer can communicate with the PLC."

RUN mode means:

    "The PLC is actively scanning and executing my ladder program."

RSLinx was doing its job.

The communications path existed.

The problem was that the emulated PLC itself was not running the ladder program.

My situation was essentially:

    RSLinx communication = GOOD
    RSLogix online        = YES
    Processor RUN mode    = NO

Because the processor was not in RUN, the ladder was not being continuously scanned.

That meant:

    Input value could change
              ↓
            BUT
              ↓
    DIGITAL_IO was not executing
              ↓
    B3 mappings were not updating
              ↓
    CONTROLS was not executing
              ↓
    Outputs were not updating

Once I placed the emulator back into **RUN**, the program immediately began working again.

---

## The Biggest Clue

The biggest clue was:

> The input value could change, but nothing downstream was reacting.

Another major clue was that ladder logic that should have shown active green power flow was not showing it.

That gives me a useful troubleshooting shortcut:

    Input changes
          +
    Downstream logic does not react
          +
    Expected green power flow is missing
          ↓
    CHECK RUN / PROGRAM MODE

Before assuming my ladder logic is wrong, I should first verify that the processor is actually executing the program.

---

## Troubleshooting the Input Side

For an input, I can follow:

    Physical Condition
          ↓
    I: Input
          ↓
    B3 Mapped Input
          ↓
    Control Logic

If:

    I: changes
    B3 does not

then I know the signal stopped somewhere between the physical input and the internal mapped bit.

Possible things to investigate include:

- Is the processor in RUN?
- Is the DIGITAL_IO routine being scanned?
- Is the correct JSR executing?
- Is the mapping rung correct?
- Is another rung writing to the same B3 address?

---

## Troubleshooting the Output Side

For an output, I can follow:

    Control Logic
          ↓
    B3 Command
          ↓
    O: Output
          ↓
    Physical Device

If:

    B3 command changes
    O: does not

then I investigate the output mapping.

If:

    O: changes
    Physical device does not

then the PLC logic has probably done its job and I need to investigate farther into the field side:

- Output module
- Wiring
- Relay
- Contactor
- VFD
- Motor
- Power
- Device fault

If nothing in the ladder is updating at all, one of my first checks should now be:

    IS THE PROCESSOR ACTUALLY IN RUN?

---

## What I Learned About Signal Ownership

One of the biggest concepts I learned was that a PLC value may be continuously controlled by another instruction.

A bit does not necessarily belong to me just because I can click it.

For example:

    I:0/0
      ↓
    OTE
      ↓
    B3:0/0

The OTE owns the state of `B3:0/0` every time that rung is scanned.

If I manually change `B3:0/0`, I am fighting the program.

The next scan can simply overwrite my change.

That changed the question I ask while troubleshooting.

Instead of:

> Why won't this bit stay ON?

I now ask:

> What instruction owns this bit?

Then:

> What controls that instruction?

Then:

> What is the root value at the beginning of this chain?

---

## What I Learned About Communication vs Execution

I also learned that these are separate things:

    COMMUNICATION
    EXECUTION

RSLinx allowing me to see the PLC does not automatically mean the PLC is running the program.

I can have:

    Communication = YES
    Online         = YES
    Ladder Scan    = NO

That is exactly what happened after restarting the VM.

The emulator needed to be placed back into RUN.

My new mental model is:

    RSLinx / Online
          =
    I can communicate with the PLC

    RUN Mode
          =
    The PLC is actually executing my program

---

## Final Mental Model

The biggest takeaway from this project is understanding the complete order of operations:

    SOURCE
      ↓
    PHYSICAL INPUT
      ↓
    INPUT MAPPING
      ↓
    CONTROL LOGIC
      ↓
    OUTPUT COMMAND
      ↓
    OUTPUT MAPPING
      ↓
    PHYSICAL OUTPUT
      ↓
    MACHINE RESPONSE

And behind the entire chain:

    PLC MUST BE IN RUN

My troubleshooting process is now:

1. Identify what should happen.
2. Find the root value that starts the event.
3. Change or simulate the root value instead of randomly changing downstream bits.
4. Follow the signal through each stage.
5. Find the exact point where the expected state stops changing.
6. Investigate that point.
7. If the entire ladder appears frozen, verify the processor is actually in RUN.

What started as a simple generator Start/Stop program became a much better lesson in PLC scan behavior, I/O mapping, signal ownership, order of operations, forcing/testing, and systematic troubleshooting.
```
