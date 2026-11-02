# Studio 5000: Local and Remote I/O Configuration

For this assignment, I configured local and remote ControlLogix I/O, then created alias and base tags. I wanted to connect what I was seeing in Studio 5000 to the physical hardware and understand how the addressing fits together.

## Questions I Explored

- How do the local and remote chassis fit together, and what is each Ethernet module responsible for?
- How does the I/O tree represent physical hardware and network connections?
- Why does the same Ethernet module appear more than once in the tree?
- How does chassis size differ from a module’s slot number?
- Which modules meet the discrete and analog I/O requirements?
- What does each part of `Local:1:I.Data.0` represent, and how does the address change for an output?
- Which tags reference physical I/O, and which hold their own values?
- Does entering a tag directly in the table create it the same way as the New Tag dialog?

<img width="1440" height="685" alt="Screenshot 2026-11-01 at 10 25 14 PM" src="https://github.com/user-attachments/assets/4f0bf127-e450-4f4b-84f6-da108f539e04" />


## Hardware Configuration

I created `Module2_Assignment` with a 1756-L71 controller and two 7-slot chassis.

| Chassis | Slot | Module | Name |
|---|---|---|---|
| Local | 0 | 1756-L71 | Module2_Assignment |
| Local | 1 | 1756-IB16 | Local_DI |
| Local | 2 | 1756-OB16E | Local_DO |
| Local | 3 | 1756-EN2T | Local_EN2T |
| Remote | 0 | 1756-EN2T | Remote_EN2T |
| Remote | 1 | 1756-IB16 | Remote_DI |
| Remote | 2 | 1756-IF8 | Remote_AI |

The local EN2T uses `10.10.10.11`, and the remote EN2T uses `10.10.10.10`.

The local controller runs the program. The two Ethernet modules provide the communication path between that controller and the remote I/O. The remote chassis doesn't need its own controller for this configuration.

## Making Sense of the I/O Tree

The Ethernet section raised the most questions for me because it doesn't follow the same simple layout as the local rack.

What clicked was that the tree shows both **physical module placement and communication paths**. An Ethernet module can appear in the network view and inside its chassis without representing two separate cards.

That explained the repeated `Local_EN2T` entry: both entries represent the same physical module in local slot 3.

I also separated chassis size from module position. A 7-slot chassis has positions 0–6; the slot setting identifies where a particular module sits.

## Connecting Tags to Hardware

An address like `Local:1:I.Data.0` identifies a specific input point:

| Part | Meaning |
|---|---|
| `Local` | The chassis containing the controller |
| `1` | Module slot |
| `I` | Input data |
| `Data.0` | Input data bit 0 |

For an output, I use `O` and the slot number of the output module.

I created these aliases:

| Alias | Hardware address |
|---|---|
| Valve1_is_open | Local:1:I.Data.0 |
| Valve1_is_closed | Local:1:I.Data.1 |
| Open_Valve1 | Local:2:O.Data.0 |

These aliases give existing I/O bits meaningful names. They refer to the same underlying values.

I also created two base tags:

| Tag | Data type |
|---|---|
| HMI_Start_Request | BOOL |
| Tank_Level | INT |

These have their own storage and aren't automatically connected to physical I/O. Calling a tag `Tank_Level` doesn't connect it to the analog module; that connection would need to be made separately.

I entered the base tags directly in the Edit Tags table. That also creates the tags without opening the New Tag dialog.

## What I Took From This

The useful connection for me was understanding how the hardware configuration, network structure, and tag addresses relate.

The chassis and slots describe where the modules are installed. The Ethernet branches describe how devices are reached. The module tags expose their data, and aliases let me use names that describe what those signals mean in the application.
