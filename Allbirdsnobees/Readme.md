# All Birds no Bees

**Download link (Google Drive)**: [LINK](https://drive.google.com/drive/folders/16S9PCaF4yroioKHA_lpC_F284IPq_Tq8?usp=share_link)<br>

![AllbirdsnobeesTitle](/Assets/AllbirdsnobeesTitle.png)

All Birds no Bees was a solo introductory Unreal game to get acquainted with developing in Unreal Engine. It is a 3D puzzle game where you place bird statues on pedestals according to provided rules of increasing difficulty. In this project, besides developing the puzzles, I spent a lot of time exploring different graphical features and playing with light and shadows.<br>
<br>
The game was entirely made using Unreal blueprints.
## Main features
### Puzzle design
The puzzle design in the game is relatively simple, merely asking the player to use the exclusion method to place statues of birds on the correct pedestals. To create the puzzles I worked backwards from the solution and created and eliminated clues until it was possible to solve the puzzle without having the solution stare you in the face.<br>

![image](https://github.com/CarlZeidler/Portfolio/assets/113012261/50b92e85-3e9e-44dd-92c2-5f3b4961ad11)

### Environment design
This game was a study in environment and graphic fidelity as much as gameplay functionality. I initially wanted to have a much more enclosed space for the player to be in, but after playing with some of the landscape settings I realized that having an outside world would add to the experience. I had planned to have an ongoing torrent of rain outside, but had to scrap it due to time limitations.
This project taught me a lot about the limitations of hardware and necessity of scoping down the looks of the game.

![image](https://github.com/CarlZeidler/Portfolio/assets/113012261/1a78c4be-02c5-4122-a017-6104ce117fea) ![image](https://github.com/CarlZeidler/Portfolio/assets/113012261/3511a43e-3a42-4ecc-a81c-cce12937b2e6)

### The statues
When creating the actual functionality of the game, I had to implement a way for the statues to "transfer" to the pedestal. The pedestal had to not just know if it had a statue sitting on it, but also to know if it was the correct statue for the puzzle.<br>

![image](https://github.com/CarlZeidler/Portfolio/assets/113012261/851be80f-8c86-4f54-9d90-7143517c6a74)

[Bird statue blueprint](https://blueprintue.com/blueprint/xplyzn2i/)<br>
[Pedestal blueprint](https://blueprintue.com/blueprint/s7npw5np/)<br>

## The UI
While the UI is fairly simple, I am particularly satisfied with how the on-screen prompt for picking up/interacting works. It raycasts from the middle of the screen in order to determine what the player is looking at and changes the tooltip accordingly. In order to make the tooltip change color depending on the color of the statue the player looking at, I had to create a data table and change
the rich text UI element by converting the enum that determines the statue's color to a string and then compare that string the color's entry in the table. It took a bit of sleuthing to figure out but I am very happy with the result.

![image](https://github.com/CarlZeidler/Portfolio/assets/113012261/df2b839a-48b2-4b4f-941b-92330b7c0c47) 
![image](https://github.com/CarlZeidler/Portfolio/assets/113012261/5787af79-a642-4295-94e9-3c80f07bdfb9)

[HUD Blueprint](https://blueprintue.com/blueprint/0edar1jz/)
