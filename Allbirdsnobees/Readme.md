# All Birds no Bees

![AllbirdsnobeesTitle](https://github.com/CarlZeidler/Portfolio/assets/113012261/31c368d9-2d29-4374-a298-7fd2de5d2ebd)

All Birds no Bees was a solo introductory Unreal game to get acquainted with developing in Unreal Engine. It is a 3D puzzle game where you place bird statues on pedestals according to provided rules of increasing difficulty. In this project, besides developing the puzzles, I spent a lot of time exploring different graphical features and playing with light and shadows.<br>
<br>
The game was entirely made using Unreal blueprints.
## Main features
### Puzzle design
The puzzle design in the game is relatively simple, merely asking the player to use the exclusion method to place statues of birds on the correct pedestals. To create the puzzles I worked backwards from the solution and created and eliminated clues until it was possible to solve the puzzle without having the solution stare you in the face.<br>

//IMAGE//

### Environment design
This game was a study in environment and graphic fidelity as much as gameplay functionality. I initially wanted to have a much more enclosed space for the player to be in, but after playing with some of the landscape settings I realized that having an outside world would add to the experience. I had planned to have an ongoing torrent of rain outside, but had to scrap it due to time limitations.
This project taught me a lot about the limitations of hardware and necessity of scoping down the looks of the game.

//IMAGE//

### The statues
When creating the actual functionality of the game, I had to implement a way for the statues to "transfer" to the pedestal. The pedestal had to not just know if it had a statue sitting on it, but also to know if it was the correct statue for the puzzle.<br>

//IMAGE//

//BLUEPRINT//

## The UI
While the UI is fairly simple, I am particularly satisfied with how the on-screen prompt for picking up/interacting works. It raycasts from the middle of the screen in order to determine what the player is looking at and changes the tooltip accordingly. In order to make the tooltip change color depending on the color of the statue the player looking at, I had to create a formatting list and change
the rich text UI element by converting the enum that determines the statue's color to a string and then compare that string the color's entry in the list. It took a bit of sleuthing to figure out but I am very happy with the result.

//IMAGE//

//BLUEPRINT//
