# Studio 5000 Emulator Upload and Download

I used my `Module2_Assignment` project to practice downloading to an emulator and uploading it back into Logix Designer. My teacher was using an older version, so I wanted to figure out which settings mattered on my setup.

## Questions I Explored

- Why could I see the emulator in Who Active while my project still showed a 1756-L71?
- Did my controller need to be in slot 1 like my teacher’s, and did the COM3 error matter?
- What does “No Controller” mean, and when would I use Upload versus Go Online?

<img width="1075" height="568" alt="Screenshot 2026-11-02 at 11 44 19 PM" src="https://github.com/user-attachments/assets/9a3b7ebf-3bc6-4346-b9c9-b7b306a2cb88" />
<img width="603" height="459" alt="Screenshot 2026-11-02 at 11 43 45 PM" src="https://github.com/user-attachments/assets/0c213e2a-7425-4d73-914f-13b469ffde9f" />

## What I Did

I changed the project controller from a 1756-L71 to an Emulate 5570 and used my v34 emulator in slot 0. I connected through FactoryTalk Linx at `Backplane\0`. My first download stopped because I had an empty rung. Once I removed it, the download completed with zero errors.

I closed the project to get to the “No Controller” screen, then selected the emulator in Who Active and opened the Upload dialog. Logix Designer found my saved `.ACD` file and gave me the option to go online with it.

## What Clicked for Me

Seeing the emulator in Who Active told me I could reach it. Controller Properties told me what my project was set up to run on. Once I separated those two things, it made sense to use my own emulator slot and the virtual backplane. The COM3 error came from a separate serial driver.

Download sends the project to the controller. Upload reads it back from the controller. Go Online lets me connect using the matching project file I already have.

## What I Need to Review

I want to be able to trace the connection the same way I’d trace a wire on a drawing. I want to know where I’m starting, what carries the data, and what’s on the other end before I start adding drivers.

In this exercise, I started in Logix Designer and used FactoryTalk Linx to reach my simulated controller through the virtual backplane. `Backplane\0` meant virtual rack, slot 0. My teacher used the virtual backplane driver named `AB_VBP-1` and selected his emulator in slot 1. RSWho showed what RSLinx Classic could see. Who Active was where I picked the controller I wanted Logix Designer to connect to.

The COM3 error came up when I tried a serial driver. COM3 would have been the computer’s end of a serial connection. My emulator was already reachable through the virtual rack, which is why I could download even though the serial driver couldn’t open that port.

The order I want to get down is to find the controller, trace the connection, check that my project is set up for it, and then decide whether I’m sending the project, reading it back, or going online.
