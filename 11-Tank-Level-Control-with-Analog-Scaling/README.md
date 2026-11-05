# Climb the Ladder: Tank Level Control

I’m working through a tank-fill exercise in Studio 5000. So far I’ve set up the rack, configured the tank’s analog input, and worked through how to tie the input and output to tags I can use in ladder logic.

## What I set up

| Slot | Module | What I’m using it for |
| --- | --- | --- |
| 0 | Emulate 5570 | Controller |
| 1 | 1756-OB16E (`Local_DO`) | Output point 11 for the fill valve |
| 2 | 1756-IB16 (`Local_DI`) | Digital inputs for the worksheet |
| 3 | 1756-IF8 (`Local_AI`) | Input channel 2 for the tank level |

## What I was figuring out

### Why does the output card say 10–31.2 VDC if the exercise says 24 VDC?

The catalog number matches, and 24 VDC is inside the range Studio 5000 shows.

### Why did I enter 10 twice for the analog scaling?

**High Signal** is what comes into the card. **High Engineering** is what the card reports to the PLC.

With both ends set to `0 → 0` and `10 → 10`, a 5 V input comes through as `5.0`.

If I wanted the card to do the conversion, I could map 10 V to 480 gallons. For this exercise, I’m doing that math in ladder.

### Are a point and a channel both locations on a card?

Yes. The slot tells me which card. The point or channel tells me which I/O location on that card.

- **Point 11** is an on/off output.
- **Channel 2** gives me a voltage reading.

### Why did the alias fill in the data type?

The alias gets its type from what it points to.

| Alias tag | Points to | Data type |
| --- | --- | --- |
| `Fill_Valve` | `Local:1:O.Data.11` | `BOOL` |
| `Tank_Level_Raw` | `Local:3:I.Ch2Data` | `REAL` |

`Fill_Valve` represents an on/off state. `Tank_Level_Raw` can hold a reading like `5.2` V.

## Where I left off

Next I’ll write the rung that converts volts to gallons. Then I’ll use that value to start filling below 80% and keep filling until the tank reaches 90%.
