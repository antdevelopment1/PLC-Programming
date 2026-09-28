# Pressure Receiver Pump Control & Indicator Logic

## 🧠 How My Understanding of the Problem Evolved

The most important part of this project was not simply arriving at working ladder logic. It was discovering a repeatable way to reason from a written process description into explicit control logic, then learning where that reasoning was sufficient and where additional control concepts were required.

My understanding developed in three major stages:

1. Use a truth/state table to derive explicit logic from the process requirements.
2. Use repeated states in that table to determine when the PLC must remember previous process history.
3. Recognize that logically correct states still do not account for the behavior of real physical inputs over time.

---

## 1. Using a Truth Table to Turn Process Requirements Into Explicit Logic

I did not begin by guessing which ladder instructions I should use.

I first translated the written process requirements into a truth/state table.

The required sequence was:

| Process State | Low | High | Pump | Indicator |
|---|---:|---:|---:|---:|
| Below 90 PSI | 0 | 0 | ON | OFF |
| Rising above 90 PSI | 1 | 0 | ON | ON |
| At / above 110 PSI | 1 | 1 | OFF | ON |
| Falling below 110 PSI | 1 | 0 | OFF | ON |
| Falling below 90 PSI | 0 | 0 | ON | OFF |

This was the first major lesson from the project.

The table gave me a way to take a process description written in English and make the required logic explicit.

Instead of thinking:

> "What ladder rung should I write?"

I could first ask:

> "For this exact combination of inputs, what should each output be?"

That gave me a repeatable reasoning process:

    Written Process Requirement
            ↓
    Identify Physical Inputs
            ↓
    Identify Required Outputs
            ↓
    Enumerate Process States
            ↓
    Assign Explicit 0 / 1 Conditions
            ↓
    Translate Those Conditions Into Ladder Logic

The truth table therefore became more than a way to test the finished program.

It became a design tool.

It allowed me to derive the logical requirements before deciding which PLC instructions should implement them.

After building my ladder logic, I then used the same sequence to verify that every condition required by the assignment was actually met:

    Below 90 PSI
    LOW = 0
    HIGH = 0
    PUMP = ON
    INDICATOR = OFF

    Rising above 90 PSI
    LOW = 1
    HIGH = 0
    PUMP = ON
    INDICATOR = ON

    At / above 110 PSI
    LOW = 1
    HIGH = 1
    PUMP = OFF
    INDICATOR = ON

    Falling below 110 PSI
    LOW = 1
    HIGH = 0
    PUMP = OFF
    INDICATOR = ON

    Falling below 90 PSI
    LOW = 0
    HIGH = 0
    PUMP = ON
    INDICATOR = OFF

Every required state produced the expected result.

That confirmed that the Boolean logic I had derived from the table was correct.

---

## 2. The Truth Table Exposed When Memory Was Required

The next discovery came directly from the table itself.

I noticed that the same input combination appeared more than once:

    LOW = 1
    HIGH = 0

At first, duplicate input combinations looked like something I might simply combine.

But these two rows required different pump outputs.

While pressure was rising:

    LOW = 1
    HIGH = 0
    PUMP = ON

Later, while pressure was falling:

    LOW = 1
    HIGH = 0
    PUMP = OFF

That was important.

If the PLC looked only at the current inputs, these two situations were identical:

    LOW = 1
    HIGH = 0

There was no Boolean expression using only those two current inputs that could simultaneously determine:

    PUMP = ON

in one situation,

and:

    PUMP = OFF

in the other.

That meant I was missing information.

The missing information was:

> What happened previously?

This was how I discovered the need for process memory.

The truth table itself exposed it.

I could now use a general rule:

> If identical current inputs require different outputs depending on how the process arrived there, the current inputs are not enough. Some previous state must be preserved.

For the pump:

    Current Inputs
          +
    Previous Pump / Process State
          ↓
    Correct Pump State

This led directly to the seal-in / hysteresis logic.

When pressure was below 90 PSI, the pump started.

Once the pump was running and pressure crossed 90 PSI, the original start condition disappeared.

However, the pump could not stop there.

It needed to remember:

> "I was already started."

The pump's own control bit therefore became part of the holding path.

Conceptually:

    Low-pressure condition
            ↓
    Pump starts
            ↓
    Pump's own state becomes TRUE
            ↓
    Pump state holds itself ON
            ↓
    Pressure continues rising
            ↓
    High-pressure condition occurs
            ↓
    Holding path is broken
            ↓
    Pump stops

This was no longer just Boolean input/output logic.

It was stateful control.

The system had history.

That was also how I began to understand hysteresis.

The pump does not use one threshold for both start and stop.

It starts below 90 PSI and stops at 110 PSI.

Between those two thresholds, the current pressure-switch states alone do not tell the complete story.

The pump's existing state matters.

---

## 3. Learning When Memory Is NOT Required

After discovering why the pump needed memory, I initially applied similar retained-state thinking to the indicator.

My first indicator implementation used OTL / OTU instructions.

It worked.

It satisfied every state in the required test sequence.

However, reviewing the truth table again exposed an important difference between the pump and the indicator.

For the pump:

    LOW = 1
    HIGH = 0

could mean:

    PUMP = ON

or:

    PUMP = OFF

depending on previous history.

Therefore the pump requires memory.

For the indicator, however:

    LOW = 0
        ↓
    INDICATOR = OFF

and:

    LOW = 1
        ↓
    INDICATOR = ON

The desired indicator state does not change depending on whether pressure is rising or falling.

Its current state can be derived directly from the current process condition.

Therefore:

    Pump
    =
    Current Inputs + Previous State

while:

    Indicator
    =
    Current Process State

This gave me another rule:

> Do not use memory simply because an output needs to stay ON.

Use memory when the required output cannot be determined from the current process state alone.

That led me to refactor the indicator.

Instead of:

    Set condition
        ↓
    OTL
        ↓
    Remember ON

and later:

    Reset condition
        ↓
    OTU
        ↓
    Remember OFF

the indicator could simply use:

    LOW_PRESSURE_SWITCH
            ↓
    OTE PRESSURE_IND_LIGHT

If LOW is TRUE, the indicator is ON.

If LOW is FALSE, the indicator is OFF.

No previous history needs to be remembered.

This was an important refinement of what I had learned about PLC memory:

> The question is not "Can I use a latch?"

The question is:

> "Does the physical process require the controller to remember something that is no longer represented by the current inputs?"

---

## 4. Comparing My Program With the Instructor's Program

At this point, my solution satisfied every required state.

The truth table had helped me:

- derive explicit logic from the written requirements
- identify hysteresis
- discover when process history mattered
- derive the need for memory
- build a seal-in circuit
- distinguish between outputs that required memory and outputs that did not
- refactor unnecessary retained state
- verify every state required by the assignment

I then compared my implementation with the instructor's implementation.

That comparison exposed an entirely different limitation.

My reasoning had been focused primarily on:

    Given these inputs,
    what should the output be?

The instructor's implementation introduced another question:

    Before changing the machine state,
    should I trust this physical input yet?

My implementation effectively assumed:

    Physical input changes
            ↓
    Boolean logic evaluates
            ↓
    Machine reacts

That is logically valid.

It also passed every condition specified by the exercise.

But it assumes the field device changes state cleanly and that every state change should be acted upon immediately.

A real pressure switch may be affected by:

- vibration
- pressure oscillation
- fluid pulsation
- mechanical contact bounce
- electrical noise
- a temporary threshold crossing

That means:

    INPUT = 1

does not automatically answer:

> "Has this physical condition existed long enough that I should allow it to change the machine?"

The truth table contains state.

It does not contain time.

For example, these two situations look identical in a Boolean table:

    LOW = 0 for 20 milliseconds

and:

    LOW = 0 for 5 seconds

Both are simply:

    LOW = 0

But physically, they may deserve very different responses.

---

## 5. Signal Qualification — Adding Time to the Decision

The instructor's implementation used TON timers to qualify the pressure-switch conditions.

The architecture was no longer:

    Sensor Changes
          ↓
    Act

It became:

    Sensor Changes
          ↓
    Detect Condition
          ↓
    TON
          ↓
    Condition Must Remain Valid
          ↓
    Timer DONE
          ↓
    Accept Condition

This introduced a new question into my reasoning:

> "Has this condition remained valid long enough that I trust it?"

For example:

    Low-pressure condition appears
            ↓
    Start 5-second timer
            ↓
    Condition remains valid?
          /       \
        NO         YES
        ↓           ↓
    Reset TON    TON.DN
                    ↓
             Accept low-pressure
                 condition

Now a momentary input change does not necessarily become a machine-state change.

The timer acts as input qualification.

This showed me that the truth table had correctly described the desired process states, but it had deliberately abstracted away the physical behavior of the sensors producing those states.

---

## 6. Conditions and Events Are Not the Same Thing

The instructor's program also introduced one-shots after the qualification timers.

Initially, this looked like additional complexity.

Comparing the architecture rung by rung helped me understand why it was there.

A timer DONE bit describes a condition:

> "Low pressure has remained valid for 5 seconds."

That condition may remain TRUE for many PLC scans.

But:

> "Start the pump."

is an event.

It only needs to happen once.

The ONS converts:

    Sustained Qualified Condition

into:

    One-Scan Event

Conceptually:

    Low-pressure condition
            ↓
    TON
            ↓
    Timer .DN
            ↓
    ONS
            ↓
    Pump Start Trigger

The timer answers:

> "Is this condition trustworthy?"

The one-shot answers:

> "Did this qualified event just occur?"

Those are different jobs.

This gave me another architecture pattern:

    CONDITION
        ↓
    QUALIFY
        ↓
    EVENT

---

## 7. The Hold Circuit Now Made More Sense

Once the start command becomes a one-scan event, the pump cannot depend on that event remaining TRUE.

Instead:

    Pump Start Event
            ↓
    Pump turns ON
            ↓
    Pump's own control bit
            ↓
    Seal-in / Hold
            ↓
    Pump continues running

A separately qualified stop event then breaks that hold:

    High-pressure condition
            ↓
    TON
            ↓
    ONS
            ↓
    Pump Interrupt
            ↓
    Break Hold
            ↓
    Pump OFF

This let me recognize the larger architecture:

    DETECT
       ↓
    QUALIFY
       ↓
    TRIGGER
       ↓
    HOLD
       ↓
    INTERRUPT

Each part has a distinct responsibility.

    XIC / XIO
    =
    What is the sensor reporting?

    TON
    =
    Has that condition existed long enough to trust?

    ONS
    =
    Did this qualified event just occur?

    Start Trigger
    =
    Request a machine-state transition

    Seal-In
    =
    Preserve the new machine state

    Interrupt
    =
    Explicitly end that state

---

## 💡 Final Takeaway

This project changed the way I approach PLC programming in several stages.

First, I learned that a truth/state table can be used to derive explicit control logic from a process description.

    Process Requirements
            ↓
    Truth / State Table
            ↓
    Explicit Input / Output Relationships
            ↓
    Ladder Logic

Then the truth table itself exposed when Boolean logic was not enough.

When the same current inputs required different outputs:

    LOW = 1
    HIGH = 0
    PUMP = ON

and later:

    LOW = 1
    HIGH = 0
    PUMP = OFF

I could prove that some previous process information had to be preserved.

That was how I discovered the need for memory.

I then learned to distinguish:

    Does the output require memory?

from:

    Can the output simply represent the current process state?

That distinction allowed me to recognize that the pump required state persistence while the indicator did not require retained memory in my original Boolean implementation.

After verifying that every condition in the assignment was satisfied, comparing my solution with the instructor's implementation exposed the next layer.

The truth table described:

> What should happen?

The instructor's timer and event architecture forced me to also consider:

> When should I trust that the physical condition has actually happened?

and:

> Is this something that remains true, or is it an event that should happen once?

My reasoning therefore evolved from:

    Process Description
            ↓
    Truth Table
            ↓
    Explicit Boolean Logic

to:

    Process Description
            ↓
    Truth / State Table
            ↓
    Identify Explicit Logic
            ↓
    Identify Repeated States
            ↓
    Determine Whether History Matters
            ↓
    Add Memory Only Where Required
            ↓
    Build Functional Ladder Logic
            ↓
    Verify Every Required Condition
            ↓
    Refactor Unnecessary Memory
            ↓
    Consider Physical Input Behavior
            ↓
    Qualify Conditions
            ↓
    Convert Qualified Conditions Into Events
            ↓
    Hold Machine State
            ↓
    Interrupt State Intentionally

The biggest lesson was therefore not simply how to use a TON, ONS, seal-in, OTL, OTU, XIC, or XIO.

It was learning how to determine why each of those tools is needed.

The truth table gave me a way to derive the explicit logic.

Repeated states taught me when memory must persist.

Refactoring taught me when memory should not persist.

Comparing my solution with the instructor's implementation taught me that a logically correct state model still has to interact with imperfect physical devices operating over time.

The larger design pattern I now recognize is:

> **Derive the State → Determine Memory → Detect → Qualify → Trigger → Hold → Interrupt**

<img width="1212" height="341" alt="Screenshot 2026-09-16 at 5 19 49 PM" src="https://github.com/user-attachments/assets/0a73f6a6-3074-4f25-a098-79eb75e5e8db" />
<img width="1208" height="518" alt="Screenshot 2026-09-16 at 5 19 57 PM" src="https://github.com/user-attachments/assets/78052336-061a-46fc-ab9b-9f2872491337" />
<img width="1203" height="547" alt="Screenshot 2026-09-16 at 5 20 03 PM" src="https://github.com/user-attachments/assets/b8172a69-41de-4528-ad5e-9400b39b7b7c" />
