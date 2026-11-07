# 12 — Climb the Ladder: Tank Level Control Continued

I’m continuing the tank-fill exercise in Studio 5000. With the hardware and I/O set up, I worked on converting the sensor voltage into gallons and controlling the fill valve.

The tank needs to start filling below 80%, continue until 90%, then stay off until it drops below 80% again.

## What I set up

| Tag | What I’m using it for |
| --- | --- |
| `Tank_Level_Raw` | Sensor voltage from Channel 2 of the slot 3 analog input module. |
| `Tank_Level_Fraction` | REAL base tag holding the voltage divided by 10. |
| `Tank_Level_Gallons` | REAL base tag holding the calculated gallons. |
| `Fill_Valve` | Alias to output point 11 of the slot 1 output module. |

## What I was figuring out

- **What does `Tank_Level_Raw` actually represent?** The module is configured so 0–10 V equals 0–10 engineering units. The value coming in is the sensor voltage. It hasn’t been converted to gallons yet.

- **Where does the math happen?** Creating the tags gives me somewhere to store the values. The calculations happen in the ladder logic. I used DIV to divide the voltage by 10, then MUL to multiply that result by the tank’s 480-gallon capacity.

- **Do I need CPT because I’m multiplying and dividing?** I wanted to avoid using CPT unnecessarily because I was thinking about memory usage. Having both operations in the formula doesn’t mean they need to happen in one instruction. I can use DIV followed by MUL, with the division’s destination becoming a source for the multiplication.

- **What is `Tank_Level_Fraction` holding?** I knew the calculation was `(7 / 10) × 480`, but the tag name wasn’t helping me connect that calculation to the instruction fields. It holds the result of `7 / 10`, which is 0.7. That’s the value the next instruction multiplies by 480. I added descriptions so I could see what each tag represents while working.

- **What actually distinguishes a base tag from an alias?** I’m still working through that distinction. Both have names I can reference throughout the program, so having a reference name isn’t what makes a tag an alias. A base tag stores its own value. An alias refers to another tag’s value, whether that’s an internal tag or hardware I/O.

- **Why REAL?** The division produces a decimal, so I chose REAL to keep that value for the multiplication. I also used REAL for the gallon result.

- **Why did `Tank_Level_Gallons` show up under local tags instead of Controller Tags?** Its scope was set to MainProgram when I created it from the rung. MainProgram-scoped tags appear under that program’s local tags. Selecting the controller as the scope puts them in Controller Tags.

- **Should all the tags be together?** For this exercise, keeping them together makes them easier to find, but it isn’t required for the math to work. MainRoutine can use both MainProgram’s local tags and controller-scoped tags. Controller-scoped tags are also available to other programs.

- **What determines the order in Controller Tags?** The existing list was sorted by name, but the new tag appeared at the bottom. The display hadn’t re-sorted that new entry yet. A single click on the header did nothing; double-clicking the Name header sorted the list. Double-clicking again switches between ascending and descending order. This only changes the display order, not the order the ladder logic executes.

## How I put it together

The scaling takes two rungs:

```text
Tank_Level_Raw ÷ 10 → Tank_Level_Fraction
Tank_Level_Fraction × 480 → Tank_Level_Gallons
```

For example, 7 volts gives a fraction of 0.7. Multiplying that by 480 gives 336 gallons.

For the valve, I started with a less-than-80% comparison. To keep filling after the level passes 80%, I added a parallel branch and used `Fill_Valve` as a seal-in.

That branch has the `Fill_Valve` contact followed by a less-than-90% comparison.

| Path | What it does |
| --- | --- |
| `Tank_Level_Fraction < 0.8` | Starts filling below 80%. |
| `Fill_Valve` on AND `Tank_Level_Fraction < 0.9` | Keeps filling until the level reaches 90%. |

At 90%, neither path is true, so the valve turns off. The seal-in contact is then off too, and filling won’t start again until the level drops below 80%.

I’m calculating gallons, but the valve rung currently compares `Tank_Level_Fraction` directly against 0.8 and 0.9.

## Where I left off

The scaling and valve-control rungs are built. I tried changing the input value, but the results weren’t updating because I was still offline and hadn’t connected the emulator.

Next is checking the final operands and testing the full fill cycle in the emulator.
