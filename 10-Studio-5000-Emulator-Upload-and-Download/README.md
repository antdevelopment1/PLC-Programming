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
