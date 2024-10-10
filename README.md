# Disc of Destiny Documentation
**Disc of Destiny** is a complete game system made in Unreal Engine 5, that aims to replicate the overall look, fell, and experience of the popular syndicated TV game show *Wheel of Fortune*. The goal of this project is to provide an easily customizable, pre-built system for game developers to create their own versions of the *Wheel of Fortune*-style experience. This game system was built with Unreal Engine 5.4 and requires no plugins outside of the code provided within the project.

## Getting Started
It is highly recommended that you at least play through a round or two of the default game in order to familiarize yourself with the general flow of gameplay and how everything fits together. To try out every single UI, here is a handy checklist of actions to try:
 * [ ] Start the default game
 * [ ] Begin the first round
 * [ ] Choose to spin the wheel
 * [ ] Land on a "Lose Turn" space after spinning the wheel
 * [ ] Land on a "Bankrupt" space after spinning the wheel
 * [ ] Choose to buy a vowel
 * [ ] Guess a correct letter
 * [ ] Guess an incorrect letter
 * [ ] Choose to solve the puzzle
 * [ ] Solve a puzzle correctly
 * [ ] Solve a puzzle incorrectly
 * [ ] Complete a round
 * [ ] Complete all 3 rounds
 * [ ] Pause the game at any point

## Core Components

### Wheel System
The physical Wheel itself is simply a cylinder (with a smaller cylinder in the center), but it's the **Pegs** that are dynamically generated, and the **Wheel Wedges** that are dynamically spawned, that provide the bulk of the functionality for the Wheel.

#### Pegs
The Pegs are capsule-shaped static meshes that are placed dynamically around the outer wedge of the Wheel, and interact with the Pointer (see below) to turn it. The number of Pegs can be changed and, because the generation of the meshes is done at construction, the result will be visible in the Viewport tab.

#### Wheel Wedges
The Wheel Wedges (or Spaces) are dynamic 2D static meshes that are generated during the `Initialize` custom event (itself called by the Game Mode when starting gameplay). The Construction event could not be used, as the Wedges themselves are actor blueprints and can only be rendered during runtime.

The contents of each Wedge are stored as the `Wheel Spaces` array in the Wheel blueprint (see **Technical Reference** » ***Wheel*** » *Wheel Variables* below). This array consists of several (24 by default) *Wheel Space Data* Structs (`S_WheelSpaceData`), each of which consisting of 6 fields:
* Space Text
	* e.g. "BANKRUPT"
* Background Color
* Text Color
* Action Type
	* *Space Action Type* Enum value (see **Technical Reference** below for more information)
* Money Value
	* Usually the cash value of a money space, but can be used for custom Space Action Types and other behavior
* Text World Size
	* Relative float value that determines how large the text is rendered atop the Wheel Wedge

#### Spinning
When the player spins the Wheel, the `Spin` event in this blueprint is called. Here, the wheel is rotated by a random rotation (between 450° and 900°), which is handled by a Timeline with a hand-crafted curve that is meant to mimic the spinning of the wheel in the actual game show on which this project is based. Upon completion, the Wheel Pointer is referenced, so that the Wheel Wedge actor blueprint can be detected via line trace. Once detected, its Space Action Type is retrieved (see **Technical Reference** » ***Data*** » *Enums* below) and this data is passed on to the Game Mode (`GM_DiscOfDestiny`), which handles what happens upon landing on each type of Wheel Wedge.

In the rare case that the Wheel Wedge cannot be detected by the Wheel Pointer, an error message is printed to both the screen and the log, and the wheel is re-spun. Note that this should only happen if, say, the Wheel Pointer is too close to the Wheel and/or points too far away to detect the Wheel Wedge below the tip of the Pointer. This has never happened in my extensive playtests of the default game, but one can never be too sure.

### Puzzle Board System
Because the **Technical Reference** section below contains extensive details about the Puzzle Board blueprint, this section will only contain a brief overview.

#### Setup
The Puzzle Board stores an array of integers (`Row Lenghts`) representing the number of tiles in each row of the board (by default, 12-14-14-12), and its length can be used to determine the number of rows total. In the Construction method, only the backplate(s) can be rendered, as they are simple planes, and not actor blueprints like each of the Letter Tiles (see below). However, even a simple backplate in the shape of the end result can be helpful in properly placing the Puzzle Board in a new or altered game level world. Without this backplate, the blueprint when placed would simply render blank, and would be quite difficult to properly place.

#### Letter Tiles
Each Letter Tile (`BP_LetterTile`) is an Actor blueprint that is rendered and placed vertically along the front of the backplate of the Puzzle Board. Each tile can have its own value, from letters to the forms of punctuation allowed by the real-world *Wheel of Fortune* game (as they are the puzzle answers imported into the project and used in the default game). Each Letter Tile contained shared functionality to:
* Populate the tile with a character
* Hide or reveal the content
* Highlight the tile during letter reveals

### Game Flow
Here is a basic outline of how the default game (as it currently is) runs:
1. Main Menu level is started by default, which creates and displays the Main Menu UI. This starts up the background music and displays the mouse cursor.
2. Upon starting the game, the Studio Set level is loaded in (with the camera facing all 3 stand-in contestant models) , and the Player Names Input UI appears, requesting the names of all 3 contestants.
3. Once entered in (the game won't let you submit without all 3 names filled in), the game checks the toggle value ("Show Contestant Intros?"). If true, a short intro of each contestant is shown, where each model animates a random "greeting" (e.g. head bowing, waving) as the camera stays on them for a few seconds. If false, it skips this sequence entirely (because I found it annoying to sit through during each playtest and, let's face it, so will you).
4. *Each game restart begins here.* Now the real fun begins. The first player is chosen by random, and the first round is started (i.e. Round Number set to 1). For each round, gameplay begins with the next step.
5. *Each round begins here.* The current Round Number is displayed in a UI that is shown for a few seconds, then a random puzzle is generated (i.e. a random row retrieved from the corresponding datatable) and fed to the Puzzle Board, which uses a custom-made algorithm to populate the tiles accordingly, hiding all tiles initially.
6. *Each turn begins here.* The Player Action Input UI is displayed (i.e. Spin/Buy/Solve) for the current player. If the player doesn't have at least $250, or if all vowels in the puzzle have been revealed, then the Buy button is disabled.
7. Depending on the action selected (Spin Wheel, Buy Vowel, or Solve Puzzle):
	1. **Spin Wheel:**
		* The player is shown spinning the wheel (using a crude animation from Mixamo that I spliced to make it look kind of like the player is spinning the wheel).
		* Depending on the Space Action Type of the Wheel Wedge detected below the tip of the Wheel Pointer:
		1. **Money:**
			* The current letter value is set to whatever the Money Amount value is, and the Guess Consonant UI is displayed, prompting the player to guess a consonant.
			* If the guess was correct, the game reveals each occurrence of the letter in the Puzzle Board, then gameplay returns to step **6** above, where the current player's turn continues.
			* If it was incorrect, the game lets the player know (Incorrect Letter Guess UI), then the Player Number stored in the GameMode is incremented (or reset to 1 if player 3 just played) and gameplay returns to step **6** above, where the next player's turn begins.
		2. **Lose Turn:**
			* The Lose A Turn UI is displayed. When the Continue button is clicked, the current Player Number increments (or flips back over to player 1 after player 3's turn) and gameplay returns to step **6** above.
		3. **Bankrupt:**
			* The Bankrupt UI is displayed, and the current player's money for the current round is reset to $0. Similar to *Lose Turn* above, the current Player Number is set to the next player and gameplay returns to step **6** above.
	2. **Buy Vowel:**
		* The Buy Vowel UI is displayed, prompting the player to select a vowel to buy.
		* When a vowel is selected, the Vowel Cost (stored in the GameMode as a variable) is deducted from the current player's money value.
		* If the guess was correct, the game reveals each occurrence of the letter in the Puzzle Board, then gameplay returns to step **6** above, where the current player's turn continues.
		* If incorrect, the game displays the same Incorrect Letter Guess UI as above, the Player Number increments to the next player, and gameplayer returns to step **6** above, where the next player's turn begins.
	3. **Solve Puzzle:**
		* The Solve Puzzle Text Input UI is displayed, prompting the player to solve the puzzle by entering in their guess of the answer.
		* If their guess is correct, then several things occur:
			* The current player's money is "banked" (i.e. added to a separate variable representing all money earned thus far) while all other player's money values are set to $0.
			* The Round End UI is displayed, showing the winner and the money they earned in the current round, as well as all money earned up until this point.
			* The Round Number is incremented. If it's > 3, that means the game is over! Skip to step **8** below, where the results are shown and choices give to restart or quit.
			* Otherwise, gameplay returns to step **5** above, where the next puzzle is generated and put up on the board.
		* If their guess is incorrect, then gameplay returns to step **5** above, where the next player's turn begins.
8. After all is said and done, and the game is over, the Game End UI is displayed, showing all player's names and total money amounts, as well as buttons giving you the option to restart or quit. If the player restarts, gameplay returns to step **4** above, where everything that needs to be is re-initialized and Round 1 begins all over again with a new puzzle and all money amounts reset to $0.

## How to Customize
### Wheel Customization
#### Adding New Wedges
If you want to add new wedges, edit existing wedges, or remove wedges from the list, look no further than the Wheel blueprint's `Wheel Spaces` array, the default value of which is what is displayed on the wheel in the default game that ships as part of the system.

Each element of the array is a *Wheel Space Data* Struct, which consists of the following fields:
* **Space Text:** The text displayed on the wedge itself. For example, "BANKRUPT" or "$750".
* **Background Color:** The background color of the wedge.
* **Text Color:** The foreground/text color of the wedge.
* **Action Type:** An Enum value that is checked against in the GameMode when a wheel spin completes and the pointer detects the wedge below its tip. The GameMode controls the functionality that occurs when each type is detected.
	* If you use one of the unused values here, remember to attach functionality to the `Switch` node, and ensure that any UI displayed at the end has an option to continue gameplay (or does so automatically after a Delay).
* **Money Value:** This is used mainly for Money type wheel spaces, but can be used for other Space Action Types as well. For example, if you implement the Free Spin type, the Money Value can be what the space turns into after landing on it.
	* To change a wheel wedge's entire appearance and value, simply access the Wheel Spaces array in the Wheel blueprint and change the relevant element in the array to the new value. The best way to do this is to add a Custom Event in the Wheel blueprint that changes the value, add a function to the GameMode Functionality Interface (`BPI_GameModeFunctionality`), then implement this in the GameMode where you can use the stored Wheel reference variable to  call the previous events. This way, you can use the built-in `Get Game Mode` node and call the event directly (which is done throughout the project for other cross-blueprint functionality).
* **Text World Size:** This is a relative value (float) that determines how large (or small) the Wheel Wedge's text appears. For a 24-space wheel, a good value for, say, *BANKRUPT* is 8 (7 for *LOSE TURN*), while a three-digit cash value (e.g. *$650*) is 15. The provided *$5000* space has a size value of 13. When customizing, you can open up the Wheel Wedge blueprint directly and play around with the values to see how it would look. Don't forget: not only should you reset your changes, but there is a smaller wheel slightly atop the larger wheel that "cuts out" a section of the bottom of each wedge, so not all of the text will be displayed if the entire radius (text height) is occupied. Try out the settings for other space values (e.g. *Bankrupt*, *$5000*) to determine roughly where your space's text should end.

#### Changing Wedges Between Rounds
Currently there is no functionality in place to have certain Wheel Wedges change their value/type between rounds. While we work on adding this feature to the system, this can be done like so:
1. Create a new struct, say `S_WheelSpaceChangeData`, that stores a Round #, Wedge Index (0-23 if using the default 24-space wheel), and New Wedge (using the existing `S_WheelSpaceData` struct to represent the new wedge to put in its place).
2. Create a new variable in the GameMode, say `Wedge Changes`, which is an array of the struct created above.
3. We'll use the *$5000* space as an example, which occupies index #11 in the default wheel space arrangement. If we wanted this wedge to change to, say, a $10,000 wedge just before the third round, we could store in this array: *3* for the round number; *11* for the wedge index; and the wedge struct consisting of "$10000" for the text, a custom background color, black for the text color, a money value of 10,000, and a text world size slightly smaller (perhaps 12 or 11).
4. Create a custom function or event within the GameMode, say `Alter Wheel Space`, which will handle changing any space in the Wheel object (via its reference) to any given value. With the given input values, access the Wheel object reference and alter the `Wheel Spaces` array accordingly. It might be best to first create the event in the Wheel blueprint itself, make the changes there, and call this event from the GameMode via the Wheel object reference.
5. At the very beginning of the GameMode's `Event Begin New Round` interface event, just after switching to the Puzzle Board camera, loop through the `Wedge Changes` array and check if each one's round number matches the current one. If so, call the `Alter Wheel Space` event/function accordingly; otherwise, do nothing. This loop check itself may even exist in its own separate event/function.

#### Wheel Design
The Wheel itself has a rather simple design, which can be easily customized with additional materials and textures to, say, add a center logo (although seeing it rotate might be dizzying for some players). Any number of Wedges can be stored in the Wheel Space data array. Your only limitation (*for now*) is that all text is displayed in one direction (vertically) as one unit in one font size.

#### Customizing the Spin
To modify the curve that animates the rotation of the wheel (and its spaces), open up the Wheel blueprint (`BP_Wheel`) and, in the `Spin` custom event, double-click the `Wheel Spin` timeline node to edit it. 

The choice to set the total time to 10 seconds was based entirely on vibes, and how the spin itself was timed during various playtests. This value can be changed, although you would have to move the nodes in the curve accordingly to match the original shape. Or, you can change this entirely and have a completely different animation curve that gives the spin any look and feel you want.

As for the default value of 720 (representing exactly 2 full spins of the wheel), this can be altered by the values passed to the `Random Float in Range` node in the `Event Spin` interface event. The Min and Max values passed in by default result in a random range of 450° and 900°, so changing these float values will alter the minimum and maximum rotation values of the wheel.

### Puzzle Customization
All puzzles are stored in the `DT_PuzzleAnswers` datatable (each row of which uses the `S_PuzzleAnswerData` struct) and were sourced from the *Wheel of Fortune* game show itself, thus ensuring that *all* puzzles in this table fit properly into the Puzzle Board's default 12-14-14-12 row structure.

> *When adding new puzzles, I would recommend using my [Puzzle Board Fit Algorithm tool](https://xylot.com/puzzlefit/) to make sure it will fit onto the board. It's a web version of the same algorithm I used in this project for fitting a puzzle answer onto the Puzzle Board.*

#### Tile Appearance
Because textures are loaded in through the material instance in use (`M_LetterTile_Screen_Filled_Instance`), customizing the tileset in use for all letters and punctuation isn't as easy as placing it onto the letter tile static mesh in the editor. It is recommended to compare your tileset to the textures currently in use to determine whether they will fit/appear properly on the Puzzle Board.

#### Puzzle Board
While technically it is possible to customize the number of tiles per row, and even the number of rows, it is not recommended, as this would most likely cause some puzzles to no longer be able to fit onto the board.

### UI Customization

#### Main Menu
The Main Menu UI has three buttons, only one of which does nothing, as it is meant to be extended for your own purposes: the Settings button. In the future, however, we plan to add an example Settings Menu UI for customization and extension. The Quit button itself can also lead to a custom confirmation dialog UI which then quits (passing an event to a handler in the UI that runs whenever OK/Confirm is clicked). This is also planned for a future release, to further aide in the customization process.

#### Display HUD
Every UI except for the Main Menu is controlled by the *Display HUD* UI, where you can find the player scores, round # header text, puzzle category footer text, pause button, and "No More Consonants/Vowels" indicators that appear when all consonants/vowels in a puzzle answer have been guessed and revealed. All of these elements can be customized as needed.

#### Pause Menu
The bottom-left "Pause Game" button demonstrates a working Pause Menu, complete with the ability to go back to the Main Menu screen, thus ending the current game.

#### Message Widgets
The user interface system in Disc of Destiny consists in part of a "Message Widget" system --- a collection of User Widgets that implement the Message Widget Functionality blueprint interface, which allows for proper implementation into the Widget Switcher on the main Display HUD UI. When customizing any of the Message Widgets, keep intact any functionality within the `Message Widget Displayed` interface event as well as any UI element(s) referenced therein. See **Technical Reference** » ***User Interface*** below for more information on each individual UI widget.

### Sounds & Music
Any of the sound cues and background music can be customized by simply adding your own and replacing the references to the default sound and music. Here is a list of every occurrence of `Play Sound 2D`:

1. Player Names Input UI --- called when the toggle widget's value is changed (separate [CC0-licensed](https://www.kenney.nl/assets/interface-sounds) On and Off sounds)
2. Puzzle Board --- called within the `Check Tile & Reveal if Highlighted` custom event, playing a custom "Ding" sound for each occurrence of a correct letter guess
3. GameMode (`GM_DiscOfDestiny`):
	1. Custom event `Show Incorrect Guess` --- called when player guesses a letter not present in the puzzle answer (plays a custom buzzer sound)
	2. Interface event `Event Letter Reveals Completed` --- called when there are no more consonants *or* no more vowels in the current puzzle answer (plays a custom jingle for each)
	3. Custom event `Player Wins Round` --- called after a player has successfully solved a puzzle and won the round (plays a custom stinger)
	4. Interface event `Event Show Incorrect Solve` --- called after a player fails to solve the puzzle correctly (plays a custom sound effect)
	5. Custom event `End Game` --- called at the end of all 3 rounds once the game has ended (plays a custom fanfare stinger)
	6. Custom event `Landed on Bankrupt` --- called when a player spins the wheel and it lands on a Bankrupt space (plays a custom sound effect)
	7. Custom event `Landed on Lose Turn` --- called when a player spins the wheeland it lands on a Lose A Turn space (plays a custom sound effect)
	8. Interface event `Event Generate & Display New Puzzle` --- called at the start of each round to generate and display a new random puzzle on the Puzzle Board (plays a custom jingle)

#### Wheel Pointer Click
The clicking sound effect that the Wheel Pointer makes is itself a Sound Cue that takes three "click" sound effects (custom made) and plays one at random, with a slight random modulation in pitch and volume. You can add more "click" sounds by importing them into Unreal Engine and adding them to the `SFX_WheelSpinClick` Sound Cue. To alter the modulation values, open up the Sound Cue and look for the Pitch Min/Max and Volume Min/Max variables.

### Additional Features
Currently, the Disc of Destiny game system only include regular run-of-the-mill rounds where the players take turns spinning, buying, or solving. In the future, we plan to implement Toss-Up, Speed-Up and Bonus Rounds, as well as a customizable Round schedule. We wanted to get the basics up and running first, demonstrating the core functionality like wheel spinning and puzzle board population before adding these additional features.

## Technical Reference
This section goes deeper into the technical details of the Disc of Destiny game system, explaining the blueprints, interfaces, and other core components that developers will interact with and possibly customize.

### Game Mode
The project's Game Mode, `GM_DiscOfDestiny`, contains the bulk of the functionality for the core gameplay. Implements the *Game Mode Functionality Interface* (`BPI_GameModeFunctionality`).

#### Game Mode Events
##### Game Mode → Gameplay Level Loaded ^BPI^
Called from the Main Menu UI (`UI_MainMenu`) after loading in the main level, this calls both the `Initialize Variables & UI` function and the `Start New Game` custom event. See below for both.
##### Game Mode → Tick
The `Tick` event for this blueprint checks for the validity of the Pointer object reference variable (set during function `Initialize Variables & UI`) and, if it's valid, calls its `Check Pointer Click` event, which itself handles the detection of conditions warranting a "click" sound effect, to mimic the show's own mechanical pointers.
##### Game Mode → Set New Puzzle
Inputs:
* Full Phrase --- *String*

Passes a given puzzle answer text to the Board object's `Set Puzzle Contents`  function. Called from only within this blueprint, from the `Generate & Display New Puzzle` interface event (see below).
##### Game Mode → Start New Game
Called only from within this blueprint (in the `Gameplay Level Loaded` interface event above), this event adds both the Display HUD (stored as the `HUD` variable) and the Fade Overlay UI (retrieved from the GameInstance. After a short ½-second delay (to allow everything to finish loading in properly), a collapsed graph `Store Actor References` is called, which stores the game level's existing Puzzle Board, Wheel, and Wheel Pointer actors as corresponding object references, for later use.

Then, all Disc of Destiny CineCameras in the level are stored in the Player Controller (`PC_DiscOfDestiny`) and the active camera is set to show all players. The Wheel's `Initialize` event is then run, and then the Fade Overlay's `Reveal` event is called, which fades the overlay out, thus "fading in" the scene. (This is also where the Restart functionality eventually picks up.) The mouse cursor is displayed, the current round number set to 1, a random starting player # chosen (between 1 and 3), and the Player Names Input UI is then displayed via the HUD object reference.
##### Game Mode → Introduce Players ^BPI^
If the "Show Contestant Intros?" toggle on the Player Names Input UI is set to **ON**, then this event will be run. All it does is call the next event with a value of 1 (the first contestant), which starts the contestant introduction sequence.
##### Game Mode → Introduce Player
Inputs:
* Player Number --- *Integer*

Handles the visual introduction of each contestant, which consists of their name fading in then out at the bottom of the screen and a character animation, randomly chosen from a group of animations sourced from Mixamo that roughly represent a greeting of some sort to the camera/crowd. This event is recursive; at the end of each call, the number passed to it is incremented by 1 and then passed to itself. At the start of the event, however, this number is checked and if it is greater than 3, then the introductions stop (as there are no more contestants) and this blueprint's `Begin New Round` interface event is called instead.
##### Game Mode → Begin New Round ^BPI^
Called from either the `Introduce Player` event above (once introductions are complete) or the Player Names Input UI (if introductions are bypassed). The Puzzle Board camera is set as the active one, then the Round Start UI is displayed while the round # is displayed at the top. The "No More Consonants/Vowels" flags are reset to false, the current letter monetary value reset to $0, and the Guessed Letters array reset to empty. After a delay of 3 seconds the Round Start UI is then hidden. A new puzzle gets generated and gameplay goes to the first (random) player.
##### Game Mode → Begin Player Turn
Called only from within, from the `Go To Next Player` interface event. The Player Actions UI (Spin/Buy Solve) is displayed for the player to choose their action. The current player is set as the active one in the Display HUD (i.e. current player's name and dollar amounts are fully opaque while the other 2 are semitransparent). The Puzzle Board camera is set to the active one, and then the Round # and Category are displayed on the HUD.
##### Game Mode → Spin Wheel ^BPI^
Called from the Spin/Buy/Solve UI when the player chooses Spin Wheel (or called again if the Wheel Pointer fails to detect a Wheel Wedge after spinning). The round # and category text are temporarily hidden while the camera changes to show all players. All 3 players are looped through, and the active player gets the "spin wheel" animation applied to them, while the other 2 have their "clapping" state set to true (a variable that the Animation Blueprint looks for as a trigger for the corresponding clapping animation. After this loop, and a delay of 1 second, the `Spin` event in the Wheel object is called, with a random float between 0.625 and 1.25 passed as the Modifier value (which is then used in calculating the final rotation value of the wheel).
##### Game Mode → Prompt Guess Letter ^BPI^
Inputs:
* Vowel? --- *Boolean*
* Ltr Value --- *Integer*

Called when the player either lands on a cash value space, or chooses to buy a vowel. The Game Mode's internal `Current Letter Value` variable is set to the given *Ltr Value* number, then the Puzzle Board camera is set to active. Either the Guess Consonant or the Guess Vowel UI is displayed (depending on the value of the *Vowel?* flag), and the round # and puzzle category text elements are displayed again.
##### Game Mode → Process Letter Guess ^BPI^
Called once the player selects a letter, this event handles what happens afterward. If a vowel was guessed, then the game's Vowel Cost (set to $250 by default) is deducted. Regardless, the guessed letter is stored internally as the Last Guessed Letter and is added to the internal list of Guessed Letters. If the letter exists in the puzzle answer, then the Puzzle Board highlights all tiles containing that letter, then --- one by one, a second or so apart --- each tile is revealed to the player, along with the Correct Letter Guess UI. If the letter does not exist, then the internal `Show Incorrect Guess` event is triggered.
##### Game Mode → Letter Reveals Completed ^BPI^
Inputs:
* Number Of Highlighted Tiles --- *Integer*

Called once the aforementioned correct tile highlighting has completed, the total cash award value (the *Number Of Highlighted Tiles* × the *Current Letter Value*) is given to the player if a consonant was guessed (none if it was a vowel, as the value has already been deducted at this point), and then a sequence of events occurs:
1. The conditions for "No More Consonants" are checked and, if the corresponding flag hasn't already been set, it  is then set to true (so that it is only triggered once). The "No More Consonants" sound is then played, followed by the display of the corresponding text in the upper-right corner of the screen.
2. The conditions for "No More Vowels" are checked and, if the corresponding flag hasn't already been set, it is then set to true. The "No More Vowels" sound is played, followed by the display of the corresponding text in the HUD.
3. If all letters have been revealed, then the player wins this round. Otherwise, the Spin/Buy/Solve UI is shown and gameplay continues.
##### Game Mode → Update Player Amounts ^BPI^
Called whenever the player earns money for a correct letter guess (from within each Contestant actor's blueprint), lands on bankrupt (and thus has their dollar amount cleared), or at the end of each round when the winning player's money is banked and all current amounts are cleared. The dollar amount won this round and the amount won in the game so far, for each of the 3 players, are passed to the HUD so that all player dollar amount text widgets can be updated properly.
##### Game Mode → Show Incorrect Guess
Called only from within the internal `Process Letter Guess` interface event when a player guesses a letter that does not exist in the current puzzle. Shows the Incorrect Letter Guess UI and plays the custom-made "buzzer" sound effect.
##### Game Mode → Prompt Solve Puzzle ^BPI^
Called when the player chooses to Solve the Puzzle from the Spin/Buy/Solve UI. Focuses on the Puzzle Board camera, [re-]displays the round # and category text, then shows the Solve Puzzle Input UI to allow the player to enter their guess.
##### Game Mode → Process Solve Attempt ^BPI^
Inputs:
* Guess --- *String*

Called after a guess is submitted for a puzzle solve attempt. Clears any Message Widgets, then checks to see if the player entered text that matches the puzzle text (case-insensitive). If so, the `Player Wins Round` event (see next below) is triggered; otherwise, the `Show Incorrect Solve` interface event is triggered (see below).
##### Game Mode → Player Wins Round
Called when the player submits a correct guess when solving the puzzle. Banks the winner's money (see next event below), then reveals the entire puzzle to the players as the "Round Win" jingle/stinger is played and the Round End UI is displayed, from where the player can continue gameplay manually.
##### Game Mode → Bank Winner's Money
Loops through each of the 3 players and, if the selected one is the active player, their money is banked and cleared; otherwise, their money is simply cleared (unless you want to award consolation prizes or some kind of bonus for something). Afterward, the `Update Player Amounts` interface event (see above) is triggered, so that the HUD has its text values updated.
##### Game Mode → Prep For Next Round ^BPI^
Called when the player clicks "Continue" from the Round End UI. The Puzzle Board is cleared and any Message Widgets are cleared from their container. If this is the last round, then the `End Game` event (see below) is triggered and nothing else happens in this event. Otherwise, the Current Round number value is incremented by 1, the "No More..." indicators are hidden from the HUD, and `Begin New Round` is triggered (see far above).
##### Game Mode → Show Incorrect Solve ^BPI^
Called when the player either gives up solving, or submits an incorrect guess at the puzzle answer. The corresponding sound effect is played as the Incorrect Solve Guess UI is displayed. From there, the player continues gameplay manually, and the next player begins their turn.
##### Game Mode → End Game
Called after the final round has ended. All players are shown as the Game Over musical stinger (royalty- and copyright-free) is played, the round # and category text are hidden, and the Game Over UI is displayed, which allows the player to either restart the game or quit to desktop.
##### Game Mode → Restart Game ^BPI^
Called when the player chooses to start a new game after the current one has ended. Each player's "Clapping" flag is set to false (stopping any clapping animation and resetting them to pre-Game Over state), then all Message Widgets are cleared out. We wait a second, fade to black, wait a second to allow the fade to play out, reset all dollar amount values (and update the HUD's text values accordingly), then trigger `Start New Game After Restarting` (see far below)
##### Game Mode → Pause Game ^BPI^
Called whenever the "Pause Game" button (in the lower-left corner of the screen) is clicked. The Pause Menu is displayed as the game itself itself is paused.
##### Game Mode → Unpause Game ^BPI^
Called whenever "Resume" is selected from the Pause Menu. The Pause Menu is hidden and the game itself is un-paused.
##### Game Mode → Handle Wheel Space Landed On ^BPI^
Inputs:
* Space --- *Wheel Space Data (Struct)*
	* Action Type --- *Space Action Types (Enum)*
	* Money Value --- *Integer*

A `Switch` node is used to split out the functionality depending on which type of space the last spin landed on, which is determined by the Wheel Wedge type detected by the Wheel Pointer at a point past and below its tip. Currently, only three cases are handled: `Money`, `Lose Turn`, and `Bankrupt`. Many others can easily be plugged in, with the existing values planned to be handled in later updates of this game system. For *Money* space types, the Space Money Value is passed as the *Ltr Value* parameter (and *false* as the `Vowel?` flag) to the internal `Prompt Guess Letter` interface event so that the player can guess a consonant. For both *Lose Turn* and *Bankrupt*, the respective one of the two following events are triggered.
##### Game Mode → Landed on Bankrupt
Called in the previous event above, when the player lands on a Bankrupt-type space on the wheel. The "Bankrupt" sound effect is played, the current player's money (from the current round) is wiped out, and the Bankrupt UI is displayed. From there, the player can continue gameplay manually, and the next player's turn begins.
##### Game Mode → Landed on Lose Turn
Called in the `Handle Wheel Space Landed On` interface event above, when the player lands on a Lose Turn-type space on the wheel. The "Lose a Turn" sound effect is played, and the Lose Turn UI is displayed. From there, the player can continue gameplay manually, and the next player's turn begins.
##### Game Mode → Go To Next Player ^BPI^
The current player number is incremented (or set to 1 if player 3's turn just ended), and the `Begin Player Turn` event (see above) is triggered.
##### Game Mode → Set Player Names ^BPI^
Inputs:
* Player 1 Name --- *String*
* Player 2 Name --- *String*
* Player 3 Name --- *String*

Called once each player's name has been set and submitted. Sets an internal map of Integers to Strings (`Player Names`) such that the String stored in key 1 is set to *Player 1 Name*, and so on for the other two players. Then, the HUD's corresponding `Set Player Names` event is triggered, with each *Player _ Name* variable above passed along.
##### Game Mode → Close Message Widgets ^BPI^
Inputs:
* End Turn? --- *Boolean*

Called when clicking the "Continue" button to resume gameplay on the Bankrupt, Lose Turn, Incorrect Letter Guess, and Incorrect Solve Attempt UIs; or, when clicking either of the Spin, Buy, or Solve action buttons in the Spin/Buy/Solve UI. The HUD's own `Clear Message Widgets` event is triggered (*Fade Out?* set to *true*). Then, if *End Turn?* is *true*, wait a second and then trigger the internal `Go To Next Player` interface event (see above).
##### Game Mode → Generate & Display New Puzzle ^BPI^
Called only from the internal `Begin New Round` interface event (see above). The Puzzle Board is cleared, then a random (non-Bonus *for now*) puzzle answer is chosen from the datatable and stored as the *Current Puzzle* (a `Puzzle Answer Data` Struct), and the Answer is passed along to `Set New Puzzle` (see above). The puzzle's Category is converted to a user-friendly string that is then passed to the HUD's `Show Puzzle Category` event before the "New Puzzle" jingle/stinger is played.
##### Game Mode → Set Player Objects ^BPI^
Inputs:
* Player Refs --- *Map: Integers → Game Contestant Actor Object References*

Called from the Player Controller's `Store Cameras` event, which also handles the detection of each contestant's actor in the game level. The map of *Player Refs* passed along from the Player Controller here is stored as an internal variable, called *Player Objects*.
##### Game Mode → Start New Game After Restarting
Called after starting a new game once the current game has ended. Delays half a second, just in case anything needs to finish reloading, then branches into the `Start New Game` event (see above), at the point where the Fade Overlay fades us back in.

#### Game Mode Functions
##### Game Mode → Get Consonant or Vowel Message Type ^PURE^
Inputs:
* Vowel? --- *Boolean*

Outputs:
* Widget Type --- *Game UI Message Widgets (Enum)*

Returns the `PlayerGuessVowel` enum value if *Vowel?* is *true*; otherwise, `PlayerGuessConsonant` is returned. This is used to determine which "Guess" UI to show when a player chooses to either spin and guess a consonant, or buy and guess a vowel.
##### Game Mode → Get Random Puzzle ^PURE^
Inputs:
* Must be Bonus? --- *Boolean*

Outputs:
* Puzzle Row --- *Puzzle Answer Data (Struct)*

Returns a random row from the Puzzle Answers datatable (`DT_PuzzleAnswers`) with its *Is Bonus?* value matching the *Must be Bonus?* value passed into this function.
##### Game Mode → Hide Mouse
Sets the Player Controller's input mode to "Game Only" and mouse cursor visibility state to *false*.
##### Game Mode → Initialize Variables & UI
Instantiates the Fade Overlay UI and stores it into the Game Instance, where it can be used in any level. Instantiates the Display HUD and Pause Menu UIs and stores them each as a private object reference variable.
##### Game Mode → Reset All Money Values
For each Game Contestant actor object reference stored in the internal Player Objects array, its `Clear All Money Values` function is run, which does as it says for that contestant.
##### Game Mode → Set Next Active Player
Increments the internal Active Player integer counter by 1. If it's greater than 3, sets it back to 1.
##### Game Mode → Show Mouse
Sets the Player Controller's input mode to "Game and UI" and mouse cursor visibility state to *true*.
##### Game Mode → Should Alert No More Consonants? ^PURE^
Outputs:
* Alert? --- *Boolean*

Returns whether or not both: the current puzzle's consonants have all been solved; *and*, the internal flag *Alerted No More Consonants?* hasn't already been set to *true*. (Basically, returns whether or not we should now alert the players that there are no more consonants left in the puzzle to guess.)
##### Game Mode → Should Alert No More Vowels? ^PURE^
Outputs:
* Alert? --- *Boolean*

Returns whether or not both: the current puzzle's vowels have all been solved; *and*, the internal flag *Alerted No More Vowels?* hasn't already been set to *true*. (Basically, returns whether or not we should now alert the players that there are no more vowels left in the puzzle to guess.)
##### Game Mode → String Contains ^PURE^
Inputs:
* Haystack --- *String*
* Needle --- *String*

Outputs:
* Found? --- *Boolean*

Simply a (better-looking) shortcut for the String `Contains` node (with both *Use Case* and *Search from End* left set to *false*), returning its return value with *Haystack*'s value passed to its *SearchIn* input and *Needle*'s passed to *Substring*. I just disliked how big and clunky the `Contains` node looked among the other, cooler-looking nodes.

#### Game Mode Variables
##### Game Mode → Board
**Type:** Puzzle Board (`BP_PuzzleBoard`)

A reference to the single Puzzle Board actor in the game level.
##### Game Mode → HUD
**Type:** Display HUD (`UI_DisplayHUD`)

A reference to the ever-present Display HUD UI in the game.
##### Game Mode → Main Menu
**Type:** Main Menu (`UI_MainMenu`)

A reference to the Main Menu UI displayed at the start of the game.
##### Game Mode → Pointer
**Type:** Wheel Pointer (`BP_WheelPointer`)

A reference to the single Wheel Pointer actor in the game level.
##### Game Mode → Wheel
**Type:** Wheel (`BP_Wheel`)

A reference to the single Wheel actor in the game level.
##### Game Mode → Pause Menu
**Type:** Pause Menu (`UI_PauseMenu`)

A reference to the Pause Menu UI displayed when the game is paused.
##### Game Mode → Active Player
**Type:** Integer

The number of the player whose turn it currently is.
##### Game Mode → Alerted No More Consonants?
**Type:** Boolean

Whether or not we have already alerted the players in this round that the puzzle no longer contains consonants to be guessed.
##### Game Mode → Alerted No More Vowels?
**Type:** Boolean

Whether or not we have already alerted the players in this round that the puzzle no longer contains vowels to be guessed.
##### Game Mode → Current Letter Value
**Type:** Integer

The current cash award value for each correctly guessed letter. Reset to 0 at the start of each round (internal `Begin New Round` interface event).
##### Game Mode → Current Puzzle
**Type:** Puzzle Answer Data Structure (`S_PuzzleAnswerData`)

The data for the current puzzle selected for this round.
##### Game Mode → Current Round
**Type:** Integer

The current round number.
##### Game Mode → Guessed Letters
**Type:** Array of Strings

All letters guessed by any contestant in this round. Used for determining the Enabled state for each consonant/vowel button in the respective UI (i.e. any letters in this array have their input buttons disabled).
##### Game Mode → Last Guessed Letter
**Type:** String

The last letter guessed by any contestant. Used by both the Correct and Incorrect Letter Guess UIs to show which letter we are either revealing in the puzzle or telling the player was incorrect.
##### Game Mode → Number of Correct Letters
**Type:** Integer

The number of occurrences revealed in the last correct letter guess.
##### Game Mode → Player Names
**Type:** Map of Integers to Strings

The name entered in (value) for each player number (key).
##### Game Mode → Player Objects
**Type:** Map of Integers to Game Contestant Actor (`BP_Contestant`) Object References

References to each player's (key) Contestant actor object (value) that exists in the game level.
##### Game Mode → Player Text Colors
**Type:** Map of Integers to Linear Color Structures

The text color (value) for each player number (key). Set by default to red, yellow, and blue (similar to the real game show).
##### Game Mode → Last Guessed Was Vowel?
**Type:** Boolean

Whether or not the last letter guessed was a vowel.
##### Game Mode → Vowel Cost
**Type:** Integer

The cost to purchase each vowel. Set by default to 250 (dollars), to match the real game show.

### Game Instance
The project's Game Instance, `GI_DiscOfDestiny`, contains functionality and references that are needed game-wide (i.e. across levels). Implements the *Game Instance Core Functionality Interface* (`BPI_GameInstanceFunctionality`).

#### Game Instance Events
##### Game Instance → Set Intro Song Playing State ^BPI^
Inputs:
* State --- *Boolean*

Sets the internal boolean flag (`Intro Song Playing?`) that determines whether or not the intro background music is currently playing, as determined by the given *State* value.
##### Game Instance → Set Fade Overlay Widget ^BPI^
Inputs:
* Fade Overlay --- *Fade Overlay* (`UI_FadeOverlay`) Object Reference

Stores the given *Fade Overlay* references locally, as `Fade`.

#### Game Instance Interfaces (Functions)
##### Game Instance → Get Fade Overlay Widget
Outputs:
* Fade Overlay --- *Fade Overlay* (`UI_FadeOverlay`) Object Reference

Returns the internally stored Fade Overlay object reference.
##### Game Instance → Is Intro Song Playing?
Outputs:
* State --- *Boolean*

Returns the value of the internal flag (`Intro Song Playing?`) that stores whether or not the Main Menu intro song is currently playing.

#### Game Instance Variables
##### Game Instance → Intro Song Playing?
**Type:** Boolean

Whether or not the Main Menu intro song is currently playing.
##### Game Instance → Fade
**Type:** Fade Overlay (`UI_FadeOverlay`) Object Reference

A reference to the Fade Overlay UI used throughout the game to fade in and out to black.

### Player Controller
The project's Player Controller, `PC_DiscOfDestiny`, contains functionality and references that are relevant on a contestant level, as well as camera functionality.

#### Player Controller Events
##### Player Controller → Store Cameras ^BPI^
Called by the Game Mode's `Start New Game` event. Resets the internal variable storing all game level camera references (`Cameras`). Then, all actors in the game level of the type *Disc of Destiny CineCamera* (`BP_GameCineCamera`) are checked. If none are found, then every ½-second we check again until we find that they have finally loaded. These actors are then stored in the *Cameras* array.  Next, the player colors array from the Game Mode is copied over to an internal variable here. Finally, all 3 Contestant actors in the level are stored internally and then passed on to the Game Mode via its `Set Player Objects` interface function.
##### Player Controller → Switch To Camera
Inputs:
* Active Camera --- *Camera Placements (Enum)* (`E_CameraPlacements`)

Called whenever a camera switch is needed (quite often and throughout various blueprints in the project). Finds the given *Active Camera* enum value in the internal *Cameras* map (as a key). It is then set as the active camera (i.e. View Target).
##### Player Controller → Trigger Animation Type ^BPI^
Inputs:
* Player Number --- *Integer*
* Animation Type --- *Player Animation Categories (Enum)* (`E_PlayerAnimationCategories`)

Finds the Contestant actor stored with the key value matching *Player Number*, then (if found) plays a random greeting animation of the type *Animation Type* via the Contestant's `Play Random Animation` event.
##### Player Controller → Set Active Camera on Player
Inputs:
* Player Number --- *Integer*

Calls the internal `Switch to Camera` event (see above) with the player camera matching the given *Player Number* (1 to 3), or the "All Players" camera if 0 (i.e. no value) was passed.

#### Player Controller Interfaces (Functions)
##### Player Controller → Get Player Objects
Outputs:
* Player Objects --- *Map: Integers → Game Contestant Actor (`BP_Contestant`) Object References*

Returns the internal `Player Objects` variable's contents.

#### Player Controller Variables
##### Player Controller → Cameras
**Type:** Map of Camera Placements (Enum) (`E_CameraPlacements`) to Disc of Destiny CineCamera (`BP_GameCineCamera`) Object References

Stores all game camera objects in the game level, with the camera placement enum value as each key. See `E_CameraPlacements` under **Data** » ***Enums*** below.
##### Player Controller → Active Camera
**Type:** Camera Placements (Enum) (`BP_GameCineCamera`)

Stores the camera placement that is currently the active view.
##### Player Controller → Player Objects
**Type:** Map of Integers to Game Contestant Actor (`BP_Contestant`) Object References

Stores a reference to each contestant actor present in the game level.
##### Player Controller → Player Text Colors
**Type:** Map of Integers to Linear Color Structures

Stores the text color for each player.

> **NOTE:** *Do* not *use this to set the color values. Do so in the Game Mode. See **Game Mode** above for more information.*

### Game Contestant
Each of the three players in-game are each represented by the project's Game Contestant actor blueprint, `BP_Contestant`, which has its own set of events, functions, and variables relevant to the contestant's skeletal mesh or per-player data like money and text color.

#### Game Contestant Events
##### Game Contestant → Play Random Animation
Inputs:
* Animation Type --- *Player Animation Categories (Enum)* (`E_PlayerAnimationCategories`)

Sets the internal `Is Currently Animating?` flag to *true* before playing a random animation from the list of animations stored internally in the `Player Animations` map variable, with the *Animation Type* value as the key. Once the animation has completed (or is interrupted for whatever reason) the internal flag is set back to *false*.
##### Game Contestant → Set Clapping State
Inputs:
* Clapping? --- *Boolean*

Sets the internal `Is Clapping?` flag to the same value as the *Clapping?* value passed here.

> **NOTE:** *This value is checked by the Animation Blueprint used for contestants, which switches to the Clapping animation state if this value is* true *.*
##### Game Contestant → Trigger Spin Wheel Animation
Plays the animation montage responsible for the cruddy placeholder "spinning the wheel" animation we have set for contestants (`AM_Player_Gesture_SpinWheel_Montage`).
##### Game Contestant → Award Money
Inputs:
* Money Offset --- *Integer*

Alters the contestant's current money amount by the *Money Offset* value passed in. So, if the number passed in negative, money will be taken *from* the contestant's current round dollar amount. After, the Game Mode's `Update Player Amounts` interface event is triggered, which then updates the dollar amounts shown on the Display HUD.

#### Game Contestant Functions
##### Game Contestant → Clear Action
Resets the internally stored `Current Action` to *- none -* (i.e. blank, for all intents and purposes).
##### Game Contestant → Clear All Money Values
Resets the dollar amount won in this current round to 0.
##### Game Contestant → Clear/Bank Money
Inputs:
* Bank? --- *Boolean*

If we are banking the money (i.e. *Bank?* is set to *true*), then the dollar amount won in this round is added to the dollar amount won so far. Regardless, the dollar amount won in this round is reset to 0.
##### Game Contestant → Get Money & Total ^PURE^
Outputs:
* Player Money --- *Integer*
* Player Total --- *Integer*

Returns the internal variables for the dollar amount won this round and won in total, respectively, as the values for *Player Money* and *Player Total*.
##### Game Contestant → Get Money Won Last Round ^PURE^
Outputs:
* Amount -- *Integer*

Returns the internal variable that stores the amount won in the last round (`Amount Won in Last Round`).
##### Game Contestant → Get Money Last Round & Total ^PURE^
Outputs:
* Player Money --- *Integer*
* Player Total --- *Integer*

Returns the internal variables for the dollar amount won in the last round and won in total, respectively, as the values for *Player Money* and *Player Total*.
##### Game Contestant → Get Is Currently Animating ^BPI^
Outputs:
* Animating? --- *Boolean*

Returns the internal variable storing whether the player is currently animating (beyond what the Animation Blueprint dictates) or not (`Is Currently Animating?`).
##### Game Contestant → Get Is Clapping ^BPI^
Outputs:
* Clapping? --- *Boolean*

Returns the internal variable storing whether or not the player is currently clapping (`Is Clapping?`). Called by the Contestant Animation Blueprint (`ABP_Contestant`) to detect whether the animation state itself should be entered or not.

#### Game Contestant Variables
##### Game Contestant → Player Model
**Type:** Skeletal Mesh

A reference to the blueprint's *Player Model* skeletal mesh.
##### Game Contestant → Root
**Type:** Scene Component

A reference to the blueprint itself.
##### Game Contestant → Amount Won In Last Round
**Type:** Integer

The dollar amount that this contestant has won so far in this round.

Normally, this variable matches the Dollar Amount This Round exactly. However, at the end of each round all players' dollar amounts (for the current round) are reset during the banking process, so a separate variable is needed to keep track of what the dollar value *was* before it was banked and thus cleared, to display on the End Game UI.
##### Game Contestant → Current Action
**Type:** Player Animation Categories (Enum) (`E_PlayerAnimationCategories`)

The animation type that this contestant's skeletal mesh is currently playing. The usage of an Enum allows us to define additional animations, as seen in the default value for `Player Animations` (see below).
##### Game Contestant → Dollar Amount This Round
**Type:** Integer

The dollar amount that this contestant has won so far in this round. For more information, see `Amount Won In Last Round` two entries above.
##### Game Contestant → Dollar Amount Total
**Type:** Integer

The dollar amount that this contestant has won so far in this game. When money is banked for the winning player at the end of each round, the `Dollar Amount This Round` is added to this integer value.
##### Game Contestant → Gender
**Type:** Player Genders (Enum) (`E_PlayerGenders`)

The contestant's "gender". This determines the player model used, and can be used for categorization of additional player models that you might add to your project.
##### Game Contestant → Is Clapping?
**Type:** Boolean

Whether or not the contestant's skeletal mesh should be playing the "clapping" animation.
##### Game Contestant → Is Currently Animating?
**Type:** Boolean

Whether or not the contestant's skeletal mesh is currently playing an animation montage.
##### Game Contestant → Player Animations
**Type:** Map of Player Animation Categories (Enum) (`E_PlayerAnimationCategories`) to Player Animations List (Struct) (`S_PlayerAnimationsList`)

The master list of all player animations, categorized by Player Animation Categories. For each entry to be an array of animations, the element that each key points to has to be a Struct, in this case the Player Animations List (which is simply one variable: an array of Anim Montage object references). The animations stored as the default values are all available in the project to use as needed, and will most likely be implemented into the default game in future updates.
##### Game Contestant → Player Number
**Type:** Integer

The number assigned to this contestant (1 to 3).
##### Game Contestant → Player Text Color
**Type:** Linear Color

The text color assigned to this contestant.

### Game Objects
The rest of the blueprints in this project comprise the "physical" aspects of the gameplay: the puzzle board, wheel, and all parts associated with them.

#### Letter Tile
Each of the 32 letter tiles is represented by the Letter Tile actor blueprint (`BP_LetterTile`), which implements the Letter Tile Behavior Interface (`BPI_LetterTileBehavior`), allowing for shared behavior among all tiles.

##### Letter Tile Events
###### Letter Tile → Set Tile Index
Inputs: 
* New Index --- *Integer*

Sets this tile's internal `Tile Index` to the *New Index* value passed here. This allows us to reference a specific tile as long as we know its index (0-51, assuming a 52-tile board).
###### Letter Tile → Highlight Tile ^BPI^
Sets the tile's current texture to the "highlighted" texture and marks the tile's internal `Is Highlighted?` flag to *true*.
###### Letter Tile → Reveal Letter ^BPI^
Using the internal `Tile Character` enum's display name as the key, the matching texture is pulled from the Letter Tile Character Textures datatable (`DT_LetterTileCharacterTextures`) and used for this tile's texture. Then, both the `Is Hidden?` and `Is Highlighted?` flags are reset to *false*.
###### Letter Tile → Hide Letter ^BPI^
If this tile's internal `Tile String` is set to *something*, then the "Hidden Blank" texture is applied to the tile's screen (showing a blank white tile) and the `Is Hidden?` flag is set to *true*.
###### Letter Tile → Clear Value ^BPI^
Sets the `Tile Character` enum to *- none -* (blank), the `Tile String` to an empty string, and the tile screen's texture to the "Unused With Logo" texture (the green cloudy background with the Disc of Destiny logo). Basically, this resets the tile to the default "unused" state.

##### Letter Tile Functions
###### Letter Tile → Get Letter String ^PURE^
Returns the internal `Tile String` value, which represents the letter/character stored in this tile as part of the current puzzle.
###### Letter Tile → Has Consonant? ^PURE^
Returns whether the internal `Tile String` value is a consonant using a string of consonants for comparison.
###### Letter Tile → Has Vowel? ^PURE^
Returns whether the internal `Tile String` value is a vowel using a string of vowels for comparison.
###### Letter Tile → Is Tile Character Value Set? ^PURE^
Returns whether or not the internal `Tile Character` enum value is anything other than *- none -* --- i.e., it's not blank.
###### Letter Tile → Is Tile Highlighted? ^PURE^
Returns the value of the internal `Is Highlighted?` flag --- i.e., whether or not this tile is in a "highlighted" state during the letter reveal process.
###### Letter Tile → Is Tile Revealed? ^PURE^
Returns the opposite value of the internal `Is Hidden?` flag --- i.e., whether or not this tile is "revealed" (simply the opposite of "hidden").
###### Letter Tile → Update Tile Visually
Applies the stored `Tile Screen Texture` to the Material Instance used for all letter tiles, thus updating the tile visually to reflect the desired texture at any given time.

##### Letter Tile Variables
###### Letter Tile → Tile
**Type:** Static Mesh Component

Reference to the blueprint's Tile static mesh.
###### Letter Tile → Root
**Type:** Scene Component

Reference to the blueprint itself.
###### Letter Tile → Is Highlighted?
**Type:** Boolean

Whether or not this tile is in a "highlighted" state (i.e. plain blue background), during the letter reveal process.
###### Letter Tile → Tile Index
**Type:** Integer

The index of this tile among the others in the Puzzle Board. This allows us to references specific tiles by it's index number in the array stored by the board's blueprint.
###### Letter Tile → Tile Screen Texture
**Type:** Texture

The texture used in the dynamic material applied to the tile's screen material slot.
###### Letter Tile → Tile String
**Type:** String

The character that this tile is set to within the puzzle answer, or empty if tile is unused.

> **NOTE:** *Empty ("") is treated very different than a space (" "), which represents an actual space character in a puzzle answer (i.e. the space between words).*
###### Letter Tile → Tile Character
**Type:** Letter Tile Characters (Enum) (`E_LetterTileCharacters`)

The character that this tile is set to within the puzzle answer, in Enum form. The value *- none -* is used for blank tiles. Each Enum value's display string is used as a row name in the datatables for letter string value (`DT_LetterTileCharacterStrings`) and associated tile texture (`DT_LetterTileCharacterTextures`). See below under **Data** » ***Tables*** for more information on both.
###### Letter Tile → Is Hidden?
**Type:** Boolean

Whether or not this tile is "hidden" --- i.e., there is a value set, but the tile has not been revealed yet to the players.

#### Puzzle Board
The game's Puzzle Board actor blueprint, `BP_PuzzleBoard`, handles most of the logic and functionality for the puzzle board or the individual tiles stored within.

##### Puzzle Board Events
###### Puzzle Board → BeginPlay
Since Letter Tiles are actors, we must spawn them at runtime (not at construction). All we can do during construction is instantiate all 52 tiles and set the data that we know (world location and row/position values), storing them in the internal array `Tile Contents`.

But here, we loop through this array. Taking the World Location value of each tile, we spawn the Letter Tile actor  in each position and then store a reference to it in the `Tile Contents` array. We also set each tile's `Tile Index` to the index of the loop iteration we're in.
###### Puzzle Board → Reveal Highlighted Tiles
Called after the tiles with occurrences of the correctly guessed letter have been highlighted. The next step would be to then reveal them, one by one. This event sets the internal variable `Number Of Highlighted Tiles` to 0, then triggers the next event `Check Tile & Reveal If Highlighted`, passing 0 as the first *Tile Index* to check.
###### Puzzle Board → Check Tile & Reveal if Highlighted
Inputs:
* Tile Index --- *Integer*

Called by the above event. This recursive event first sets the internal variable `Currently Checking Tile Index` to the given *Tile Index*, so we can keep track of this more easily. As long as *Tile Index* is a valid index value (i.e. it's less than 52), we run a check to see if this Letter Tile is in a "highlighted" state (i.e. it was highlighted during the letter reveal process and we must now reveal it to the players dramatically); if it is, then we increment the `Number Of Highlighted Tiles` counter and reveal the tile's content. We wait 1.5 seconds (seems to be the sweet spot when playing) before running this event recursively with the value of *Tile Index* incremented by 1.

If we have been passed a *Tile Index* value of 52, then we have checked all tiles and the Game Mode's `Letter Reveals Completed` interface event is called, with the value of `Number Of Highlighted Tiles` passed to the event parameter of the same name.

##### Puzzle Board Functions
###### Puzzle Board → Are All Consonants Solved? ^PURE^
Outputs:
* All Solved? --- *Boolean*

Goes through each tile in the `Tile Contents` array and, if the tile both contains a consonant and is hidden, then we return *false* to signify that there is at least one consonant in the puzzle that has yet to be solved. Otherwise, if we have completed the loop, that means we have not found any unsolved consonants, so we return *true* to signify that all consonants have been solved.
###### Puzzle Board → Are All Letters Revealed? ^PURE^
Outputs:
* All Revealed? --- *Boolean*

Goes through each tile in the `Tile Contents` array and, if the tile is hidden, then we return *false* to signify that there is at least one letter in the puzzle that has yet to be solved. Otherwise, if we have completed the loop, that means we have not found any unsolved letters, so we return *true* to signify that all letters have been solved.
###### Puzzle Board → Are All Vowels Solved? ^PURE^
Outputs:
* All Solved? --- *Boolean*

Goes through each tile in the `Tile Contents` array and, if the tile both contains a vowel and is hidden, then we we return *false* to signify that there is at least one vowel in the puzzle that has yet to be solved. Otherwise, if we have completed the loop, that means we have not found any unsolved vowels, so we return *true* to signify that all vowels have been solved.
###### Puzzle Board → Clear Board
Loops through each stored tile and clears its value by calling each Letter Tile's `Clear Value` interface event.
###### Puzzle Board → Does String End In Hyphen? ^PURE^
Inputs:
* String To Check --- *String*

Outputs:
* Ends In Hyphen? --- *Boolean*

Used during the process that populates the board's tiles with a given puzzle answer's text, and allows us to split up hyphenated words as though they were separate words (but *not* add a space in between) to better fit compound words and phrases. Does what it says on the tin, and returns whether or not the *String To Check* ends with the hyphen character ("-").
###### Puzzle Board → Fill String Array ^PURE^
Inputs:
* Count --- *Integer*
* Contents --- *String*

Outputs:
* Array --- *Array: Strings*

Fills a temporary array of strings with *Count* occurrences of *Contents* and returns the result.
###### Puzzle Board → Fit Phrase To Rows
Inputs:
* Phrase --- *String*

Outputs:
* Success? --- *Boolean*
* Filled Rows --- *Array: Strings*

Takes a puzzle answer (*Phrase*) and breaks it apart into an array of words (including compound word/phrase segments), then uses a simple algorithm to attempt to fit each word into each row until it either runs out of words or rows to fill. If it runs out of words, the fit algorithm has completed successfully and *Success?* is set to *true* with an array of strings, each string representing the text in a row of the puzzle board (including spaces), passed as the result. If at any point it fails, *Success?* is set to *false* and an empty array is passed as the result.
###### Puzzle Board → Get Currently Checking Tile Actor ^PURE^
Outputs:
* Tile --- *Letter Tile* (`BP_LetterTile`)

Returns a reference to the Letter Tile actor stored at the `Currently Checking Tile Index` (referenced above; defined below) of the `Tile Contents` array of tiles.
###### Puzzle Board → Get Letter Tile Actor At Index ^PURE^
Inputs:
* Index --- *Integer*

Outputs:
* Actor Ref --- *Letter Tile* (`BP_LetterTile`)

Looks up the given *Index* as the key in the `Tile Contents` array and returns the Actor Ref stored therein.
###### Puzzle Board → Get Number Of Rows In Puzzle ^PURE^
Outputs:
* Count --- *Integer*

Returns the length of the `Row Lengths` array, which stores the number of tiles in each row (i.e. `[12, 14, 14, 12]`).
###### Puzzle Board → Get Number Of Tiles On Board ^PURE^
Outputs:
* Count --- *Integer*

Returns the sum of each value stored in the `Row Lengths` array, which stores the number of tiles in each row (i.e. `[12, 14, 14, 12]`). In our case, this would be *52*.
###### Puzzle Board → Get Tile Character Enum From String ^PURE^
Inputs:
* Character Str --- *String*

Outputs:
* Character Enum --- *Letter Tile Characters (Enum)* (`E_LetterTileCharacters`)

Returns the enum value in Letter Tile Characters that has a matching Row Name in the Letter Tile Character Strings datatable (`DT_LetterTableCharacterStrings`), or *- none -* if not found (and thus an unused tile set for the tile in question).
###### Puzzle Board → Get Tile Index From Row, Pos ^PURE^
Inputs:
* Row Index --- *Integer*
* Position in Row --- *Integer*

Outputs:
* Tile Index --- *Integer*

Calculates the *Tile Index* value that would exist in the given *Row Index* at the given *Position in Row* (thus allowing us to select an individual Letter Tile if we know the row number and position within that row) and returns that value.
###### Puzzle Board → Hide Entire Puzzle
Loops through each stored tile and hides it by calling each Letter Tile's `Hide Letter` interface event.
###### Puzzle Board → Highlight All Tiles Containing Letter
Inputs:
* Letter --- *String*

Outputs:
* Number Highlighted --- *Integer*

Loops through each stored tile and checks its letter value (via its `Get Letter String` function) against the given *Letter*. If it's a match, highlight it by calling its `Highlight Tile` interface event, then increment a local variable we use to keep track of how many we have highlighted so far. Upon completion of the loop, return the local counter's value as the *Number Highlighted*.
###### Puzzle Board → Reveal Entire Puzzle
Loops through each stored tile and reveals its contents by calling each Letter Tile's `Reveal Letter` interface event.
###### Puzzle Board → Reveal Tile At Index
Inputs:
* Tile Index --- *Integer*

Retrieves the Letter Tile actor at the given *Tile Index* via `Get Letter Tile At Index` (see above) and reveals it by calling the tile's `Reveal Letter` interface event.
###### Puzzle Board → Set Puzzle Contents
Inputs:
* Full Phrase --- *String*

Takes the given *Full Phrase* (the entire puzzle answer in one string) and attempts to `Fit Phrase To Rows` (see above). All puzzles in the game should work, so the *false* condition does not currently have any functionality other than a `Print String` to both the screen and the log (in red, shown for 10 seconds).

In most (*all*, but one can never truly be absolutely 100% sure about anything other than the statement "I exist") cases, the empty rows are removed, the letters centered by adding row offsets, and each Letter Tile on the board has its value set to the corresponding letter in the puzzle answer. If a tile has a letter, it is hidden (displayed as a blank white screen); otherwise, it is displayed (e.g. punctuation). Spaces in the puzzle are shown as unused tiles.

##### Puzzle Board Variables
###### Puzzle Board → Currently Checking Tile Index
**Type:** Integer

The Tile Index we are currently checking for in the `Check Tile & Reveal if Highlighted` interface event (see above) while we are checking for the tiles we highlighted so that we can reveal them to the players one by one.
###### Puzzle Board → Number of Highlighted Tiles
**Type:** Integer

The number of tiles that have been highlighted as part of the letter reveal process (see `Check Tile & Reveal if Highlighted` above).
###### Puzzle Board → Tile Contents
**Type:** Array of Letter Tile Data (Struct) types

The contents of each tile that exists on the board (all 52 of them in the default game). See `Letter Tile Data` under **Data** » ***Structs*** below.
###### Puzzle Board → Origin
**Type:** Scene Component

A reference to the blueprint's root Scene Component.
###### Puzzle Board → Backplate Padding
**Type:** Float
**Default:** 0.05

Used in the Construction Script to generate the backplate (which acts as both a visual aid as well as an actual backplate behind the Letter Tile actors.
###### Puzzle Board → Row Lengths
**Type:** Array of Integers
**Default:** [12, 14, 14, 12]

Represents the number of tiles in each row of the board.
###### Puzzle Board → Tile Height
**Type:** Float
**Default:** 106.68

Height modifier for rendering each Letter Tile. If customizing anything about the size of each tile, pay attention to both this variable and `Tile Width` below.

> *This is sort of a "magic number", so my apologies for this vagueness in what this number defines. I plan on revisiting each model and reimporting with a different scale as (and if) needed.*
###### Puzzle Board → Tile Width
**Type:** Float
**Default:** 86.36

Width modifier for rendering each Letter Tile. If customizing anything about the size of each tile, pay attention to both this variable and `Tile Height` above.

> *This is sort of a "magic number", so my apologies for this vagueness in what this number defines. I plan on revisiting each model and reimporting with a different scale as (and if) needed.*
###### Puzzle Board → Letters
**Type:** Scene Component

A reference to the Scene Component that stores all Letter Tiles.
###### Puzzle Board → Backdrop
**Type:** Scene Component

A reference to the Scene Component that contains the backplate.

#### Wheel
The game wheel itself, `BP_Wheel`, contains the logic and functionality necessary for spinning the wheel, as well as generating the pegs and wedges on the surface.

##### Wheel Events
###### Wheel → Initialize
Called at the start of the game (only once) to initialize the spaces atop the wheel. For each Wheel Space Data structure stored in `Wheel Spaces` (see below), each wheel space's rotation is calculated and each Wheel Wedge actor is spawned and positioned properly, such that each space occupies an equal amount of real estate atop the wheel, adding up to an even 360° (to fill up the entire wheel and leave no gaps between wedges). Each Wheel Wedge is then added to the `Wedges` variable (see below).
###### Wheel → Spin
Inputs:
* Modifier --- *Integer*

Responsible for actually spinning the wheel's static mesh. Called whenever a player chooses to spin the wheel when it is their turn.

Before anything happens, the current rotation of the wheel is stored (in `Wheel Rotation Offset`), so that we can start the spin at its current rotation (and not at 0°). First, the Timeline `Wheel Spin` (see below, under *Wheel Variables*) is used to animate the Z-rotation of the wheel mesh. The *Modifier* (a float representing a factor by which we multiply the default 720° rotation represented by the return value of the Timeline node. The value between 0 and 720 is multiplied by this number, then added to the `Wheel Rotation Offset`, to give us the rotation to use at each *Update* execution path (from the Timeline node), where the wheel mesh's relative rotation is set to this value.

Concurrently with this Timeline call (as accomplished using the Sequence node), Delays and camera changes are used for cinematic effect, similar to the real world game show's production --- and for demonstration purposes as well. This is easy to customize; add new cameras if needed, and alter the Delay amounts and order of camera changes. Keep in mind that the Sequence ensures that each execution path is run one after the other without waiting for completion of the previous iteration, so keep this in mind when setting your custom Delay duration values.

Once the Timeline animation completes (set to 10 seconds), we retrieve a reference to the game's Wheel Pointer  (via the Game Mode) and react to the space that the spin landed on by calling the Pointer's `Detect Wedge And Act` event.

##### Wheel Functions
###### Wheel → Get Wedge Angle ^PURE^
Outputs:
* Angle --- *Float*

Returns the angle of the wedge (360° divided by the number of wheel spaces). In the default game there are 24 spaces on the wheel, so each wedge would have an *Angle* of 15° (360° ÷ 24).
###### Wheel → Get Wedge Location ^PURE^
Outputs:
* Location --- *Vector*

Returns a location that is at world center, with a Z-offset (height) of 0.2 units above the `Wedges` Scene Component's Z-offset, so the wedges show up *just* atop the wheel (and to avoid polygon jumbling).
###### Wheel → Is Wheel Spinning? ^PURE^
Outputs:
* Spinning? --- *Boolean*

Returns the value of the internal flag `Is Spinning?` (which is set to true during the 10-second wheel spin process).
###### Wheel → Render Outer Pegs
Adds `Number of Pegs` (see below) instances of the peg static mesh to the `Pegs` Instanced Static Mesh Component, positioned at evenly-spaced locations along (but just inside) the outer rim of the wheel mesh. This allows us to not only quickly calculate the locations of all 72 pegs (the default value of `Number of Pegs`) for you, but lets you customize this value as well. You can have more pegs, or fewer pegs, or a different mesh altogether.

##### Wheel Variables
###### Wheel → Wheel Rotation Offset
**Type:** Float

The current rotational offset of the wheel so that future spins are based on this value.
###### Wheel → Pegs
**Type:** Instanced Static Mesh Component

Parent for all wheel pegs. The static mesh used for the pegs are stored in this component's settings (not in any blueprints), so if you want to customize the appearance of the pegs, do so here.
###### Wheel → Wedges
**Type:** Scene Component

Parent for all Wheel Wedge (`BP_WheelWedge`) actors generated at runtime.
###### Wheel → Inner Wheel
**Type:** Static Mesh Component

Reference to the green inner wheel mesh, which is just a cylinder with a metallic material applied.
###### Wheel → Wheel
**Type:** Static Mesh Component

Reference to the main wheel mesh.
###### Wheel → Base
**Type:** Scene Component

Reference to the root Scene Component of this blueprint.
###### Wheel → Wheel Spin
**Type:** Timeline Component

The Timeline graph used to animate the spinning of the wheel in a realistic manner, from 0 to 720° (by default) over the course of 10 seconds (again by default).
###### Wheel → Radial Distance
**Type:** Float
**Default:** 113.0

The distance from the center of the wheel at which point the wheel pegs should be placed.
###### Wheel → Total Number of Pegs
**Type:** Integer
**Default:** 72

The total number of pegs you want to render. The default value of 72 matches that of the real world game show.
###### Wheel → Wedge Radius
**Type:** Float
**Default:** 112.0

The radius of each Wheel Wedge (basically the length of the sides). Smaller values leave a larger gap around the outer edge of the Wheel.
###### Wheel → Wheel Spaces
**Type:** Array of Wheel Space Data (`S_WheelSpaceData`) Structures

*This* is where all of the Wheel spaces are defined. Each Wheel Space Data Structure (see below, under **Data** » ***Structs***) contains the necessary information for each Wheel Wedge, and any customizations to the spaces can be done here. The Action Type in each struct is where you can set how the game reacts to the space in question.
###### Wheel → Is Spinning?
**Type:** Boolean

Whether or not the Wheel is currently spinning. This should only be true for the 10-second period during which the wheel's rotation value is controlled by the *Wheel Spin* Timeline.

#### Wheel Pointer
The Wheel Pointer, `BP_WheelPointer`, is the meat and potatoes of the wheel spin functionality. Not only is it responsible for detecting and reporting the type of space that the wheel spin has landed on, but it also rotates and plays a "click" sound effect that changes slightly each time it's played, for a realistic effect.

For the values that control the rotational thresholds for pull and release to simulate the clicking sound effect, see the *Wheel Wedge Variables* below

##### Wheel Pointer Events
###### Wheel Pointer → BeginPlay
Here we simply store the current world rotation of the `Pointer` static mesh into the `Starting Rotation` variable (see below), to act as an offset when detecting rotation compared to its starting rotation.
###### Wheel Pointer → Check Pointer Click
Called on every `Tick` of the game (see **Game Mode** » ***Game Mode Events*** above). Checks the rotation of the Pointer against the `Springback Tolerance` and `Pullaway Tolerance` values, as well as the Boolean flag we use (`Has Sprung Back?`) to determine if we need to play the click sound effect (`SFX_WheelSpinClick`) accordingly.

The `Has Sprung Back` flag starts at *true* to allow for "cocking" of the imaginary "mechanism" that is simulated using the two `Tolerance` variables (see below). The flag is set to *false* once the Pointer has been pushed past the `Pullaway Tolerance` value, then set to *true* again when the Pointer rotates back below the `Springback Tolerance` value.

You can play with these values to achieve different clicking effects and frequencies.
###### Wheel Pointer → Detect Wedge and Act
Using the `Wedge Detection Origin` Scene Component's world location as a baseline, a line trace aiming downward detects if we've hit a Wheel Wedge actor (`BP_WheelWedge`) --- which we should every single time, if the Pointer is placed correctly in the game level in comparison to the Wheel (`BP_Wheel`). If we don't hit a wheel space, then we re-spin and the process repeats itself. Otherwise, if we do, get the data for this space (via the `Get Wheel Space Data` function of the Wheel Wedge we hit. This data is passed along to the Game Mode's `Handle Wheel Space Landed On` interface event (see above), which handles the functionality that runs for each space type.

##### Wheel Pointer Functions
###### Wheel Pointer → Get Pointer Rotation ^PURE^
Outputs:
* Rotation --- *Float*

Returns the relative rotation of the Pointer compared to how it was placed in the game level (i.e. compared to `Starting Rotation`, set in `BeginPlay` above).
###### Wheel Pointer → Get Pointer Detector Origin ^PURE^
Called by the line trace that is run in `Detect Wedge and Act` above. Simply returns the world location of the Wedge Detection Origin.

##### Wheel Pointer Variables
###### Wheel Pointer → Has Sprung Back?
**Type:** Boolean
**Default:** *True*

Internal flag representing whether or not the Pointer has "sprung back". For more information on how this flag is utilized, see the `Check Pointer Click` event above.
###### Wheel Pointer → Starting Rotation
**Type:** Float

The rotation at which the Pointer lies at the start of the game. Used for calculations in Pointer rotation for the clicking sound effect.
###### Wheel Pointer → Constraint
**Type:** Physics Constraint Component

This... was a headache and a half. This component and its settings control how the Pointer rotates and reacts to being moved, as well as how quickly it releases back. Too slow and it looks like molasses has gummed up the works. Too fast and it passes right through most of the pegs. The values that this component is set to might still need further tweaking, but... there's only so much time one can spend on a particular thing before it resembles banging one's head against a wall.

Good luck with this one!
###### Wheel Pointer → Pointer
**Type:** Static Mesh Component

Reference to the teardrop-shaped pointer mesh that rotates against the Physics Constraint.
###### Wheel Pointer → Pivot Point
**Type:** Static Mesh Component

Reference to the base cylindrical mesh that sits beneath the Pointer. Mostly for show.
###### Wheel Pointer → Scene Root
**Type:** Scene Component

Reference to this blueprint's root component.
###### Wheel Pointer → Wedge Detection Origin
**Type:** Scene Component

Reference to a Scene Component, the location of which is used as the origin point for the line trace during Wheel Wedge detection (see `Detect Wedge and Act` above for more information).
###### Wheel Pointer → Springback Tolerance
**Type:** Float
**Default:** 5.0

If the Pointer's rotation falls below this value while the `Has Sprung Back?` flag is set to *false*, the "click" sound effect is played (see `Check Pointer Click` above for more information) and the flag is reset to *true*. See next entry for more information.
###### Wheel Pointer → Pullaway Tolerance
**Type:** Float
**Default:** 7.5

If the Pointer's rotation increases above this value while the `Has Sprung Back?` flag is set to *true*, then we can reset the check for whether we should play the "click" sound effect by setting this flag to *false*. See previous entry for more information. Just don't, like, go back and forth in some kind of infinitely regressing loop or anything.

#### Wheel Wedge
Each space on the wheel is represented by an instance of a Wheel Wedge, `BP_WheelWedge`. When instantiated, each Wheel Wedge has its corresponding data from the `Wheel Spaces` array (in the Wheel actor blueprint; see above) stored internally.

##### Wheel Wedge Functions
###### Wheel Wedge → Calculate Vertices
Outputs:
* Vertices --- *Array: Vectors*

Called during Construction of this Wheel Wedge when rendering wedge mesh. Starting with the center point as the first element, compile an array of vertex locations (Vectors) along the outer edge of the wedge's curve, and return this as *Vertices*.
###### Wheel Wedge → Calculate Normals
Outputs:
* Normals --- *Array: Vectors*

Called during Construction of this Wheel Wedge when rendering the wedge mesh. Returns an array of Vectors of value *(0, 0, 1)* with a count matching the number of vertices (calculated in the previous function).
###### Wheel Wedge → Calculate Triangles
Outputs:
* Triangles --- *Array: Integers*

Called during Construction of this Wheel Wedge when rendering the wedge mesh. Constructs an array of triangles in the form of triples of float values and returns this as *Triangles*.
###### Wheel Wedge → Calculate UVs
Outputs:
* UVs --- *Array: Vector 2D Structures*

Called during Construction of this Wheel Wedge when rendering the wedge mesh. Compiles an array of 2D (UV) locations required by the Create Mesh Section node.
###### Wheel Wedge → Get Center Vertex
Outputs:
* Array --- *Array: Vectors*

Simply a shorthand for an array consisting solely of a center coordinate (`(0, 0 ,0)`), used in the instantiation of the *Vertices* local variable within the `Calculate Vertices` function (see above).
###### Wheel Wedge → Get Wheel Space Data
Outputs:
* Data --- *Wheel Space Data Structure* (`S_WheelSpaceData`)

Called after spinning the wheel and upon landing on a valid Wheel Wedge. Returns a constructed Wheel Space Data variable consisting of values from this Wheel Wedge's internal variables: `Text Content`, `Color`, `Text Color`, `Action on Land`, `Cash Value`, and `Text World Size` (see below for all). Data then gets passed on to the Game Mode's `Handle Wheel Space Landed On` interface event (see above).
###### Wheel Wedge → Set Wedge Material
Called during Construction of this Wheel Wedge when rendering the wedge mesh. Creates a Dynamic Material Instance to which we then apply the background `Color` as the parameter, thus coloring the wedge mesh.
###### Wheel Wedge → Set Wedge Text and Positioning
Called during Construction of this Wheel Wedge when rendering the wedge mesh. Calculates where the Wheel Wedge's text should be positioned and how large it should be.
###### Wheel Wedge → Get Index In Wheel
Outputs:
* Index --- *Integer*

Returns the index of this tile in the wheel (0-23 in the default 24-space wheel), stored in the internal `Index In Wheel` variable (see below).

##### Wheel Wedge Variables
###### Wheel Wedge → Space Text
**Type:** Text Render Component

Reference to the component responsible for rendering the text atop the Wheel Wedge.
###### Wheel Wedge → Procedural Mesh
**Type:** Procedural Mesh Component

Reference to the actual [procedural] mesh that forms the basis for the visual aspect of this Wheel Wedge.
###### Wheel Wedge → Root
**Type:** Scene Component

Reference to the root component of this blueprint.
###### Wheel Wedge → Angle Step
**Type:** Float

Represents the increment in degrees of each segment in the *Vertices* array.
###### Wheel Wedge → Dynamic Material
**Type:** Material Instance Dynamic

Material applied to the mesh of this Wheel Wedge (just a solid color).
###### Wheel Wedge → Normals
**Type:** Array of Vectors

List of Normals generated by `Calculate Normals` above.
###### Wheel Wedge → Num Vertices
**Type:** Integer

Number of vertices/segments, calculated within `Calculate Vertices` (see above).
###### Wheel Wedge → Triangles
**Type:** Array of Integers

List of triangle values generated by `Calculate Triangles` above.
###### Wheel Wedge → UVs
**Type:** Array of Vector 2D Structures

List of UV coordinates generated by `Calculate UVs` above.
###### Wheel Wedge → Vertices
**Type:** Array of Vectors

List of Vertices generated by `Calculate Vertices` above.
###### Wheel Wedge → Action on Land
**Type:** Space Action Types (Enum) (`E_SpaceActionTypes`)
**Default:** *Money*

Stores the Space Action Type value for this Wheel Wedge, which determines what happens when the player's spin lands on it.
###### Wheel Wedge → Angle
**Type:** Float
**Default:** 15.0

Populated during spawning of Wheel Wedge actors by the Wheel at runtime. Represents the angle of this Wheel Wedge's arc.
###### Wheel Wedge → Cash Value
**Type:** Integer

The monetary value assigned to correct consonant letters. Only applies to *Money*-type Wheel Wedges --- or any other you might decide to apply it to.
###### Wheel Wedge → Color
**Type:** Linear Color

Background color set for this Wheel Wedge.
###### Wheel Wedge → Granularity
**Type:** Float
**Default:** 72.0

Number of segments total. Used in calculation of Vertices (see `Calculate Vertices` above).
###### Wheel Wedge → Radius
**Type:** Float
**Default:** 112.0

Populated during spawning of Wheel Wedge actors by the Wheel at runtime. Radius, in units, of this Wheel Wedge.
###### Wheel Wedge → Rotation Offset
**Type:** Float

Populated during spawning of Wheel Wedge actors by the Wheel at runtime. Rotation offset, in degrees, by which this Wheel Wedge is rotated, so that each wedge occupies a different space on the wheel.
###### Wheel Wedge → Index in Wheel
**Type:** Integer

Index of this Wheel Wedge in the Wheel (0-23 for the default 24-space wheel).
###### Wheel Wedge → Text Color
**Type:** Linear Color
**Default:** *(0, 0, 0)* [black]

Populated during spawning of Wheel Wedge actors by the Wheel at runtime. Color of the text rendered atop this wedge.
###### Wheel Wedge → Text Content
**Type:** Text
**Default:** "$500"

Populated during spawning of Wheel Wedge actors by the Wheel at runtime. Content of the text rendered atop this wedge.
###### Wheel Wedge → Text Radial Offset
**Type:** Float
**Default:** 112.0

Radial offset in degrees of the text renderer. Same as the `Radius` value (see above).
###### Wheel Wedge → Text World Size
**Type:** Float
**Default:** 16.0

World size, in units, of the rendered text.

### Blueprint Interfaces
To avoid resource-intensive references to the custom Game Mode, Game Instance, and Player Controller blueprints used in this project, Blueprint Interfaces are utilized instead. We also use BPIs for shared functionality that must be called on a number of separate objects --- for example, Contestants, Message Widgets, and Wheel Wedges. The full list is below.

#### Game Contestant Functionality Interface --- `BPI_ContestantFunctionality`
* `Get Is Clapping`
* `Get Is Currently Animating`

#### Game CineCamera Interface --- `BPI_GameCineCameraFunctionality`
* `Get Camera Label`
* `Get Self Reference`

#### GameMode Core Functionality Interface --- `BPI_GameModeCoreFunctionality`
* **Events**
	* `Begin New Round`
	* `Gameplay Level Loaded`
	* `Handle Wheel Space Landed On`
	* `Letter Reveals Completed`
	* `Pause Game`
	* `Prep For Next Round`
	* `Prompt Guess Letter`
	* `Prompt Solve Puzzle`
	* `Process Letter Guess`
	* `Process Solve Attempt`
	* `Restart Game`
	* `Show Incorrect Solve`
	* `Spin Wheel`
	* `Unpause Game`
* **Gameplay**
	* `Get Current Letter Value`
	* `Get Current Round #`
	* `Get Vowel Cost`
	* `Get Wheel Pointer Object`
* **Players**
	* `Get All Player Amounts & Totals`
	* `Get All Player Last Amounts & Totals`
	* `Get Current Player #`
	* `Get Current Player Ref`
	* `Get List of Guessed Letters`
	* `Get Player Name & Color`
	* `Get Player Text Colors`
	* `Go to Next Player`
	* `Introduce Players`
	* `Set Player Names`
	* `Set Player Objects`
* **Puzzle**
	* `Generate & Display New Puzzle`
	* `Get Last Guessed Letter`
	* `Get Number of Correct Letters`
* **User Interface**
	* `Close Message Widgets`
	* `Get HUD`
	* `Update Player Amounts`

#### Letter Tile Behavior Interface --- `BPI_LetterTileFunctionality`
* **Behavior**
	* `Clear Value`
	* `Hide Letter`
	* `Highlight Tile`
	* `Reveal Letter`
	* `Set Tile Value`

#### Message Widget Functionality Interface --- `BPI_MessageWidgetFunctionality`
* **Events**
	* `Message Widget Displayed`

#### GameInstance Core Functionality Interface --- `BPI_PersistentGameCoreFunctionality`
* **Main Menu**
	* `Is Intro Song Playing?`
	* `Set Intro Song Playing State`
* **User Interface**
	* `Get Fade Overlay Widget`
	* `Set Fade Overlay Widget`

#### Player Controller Functionality Interface --- `BPI_PlayerControllerFunctionality`
* **Animation**
	* `Trigger Animation Type`
* **Camera**
	* `Set Active Camera`
	* `Set Active Camera on Player`
* **Events**
	* `Store Cameras`
* **Players**
	* `Get Player Objects`

#### Wheel Wedge ID Interface --- `BPI_WheelWedgeIdentification`
* `Get Index in Wheel`

### Data

#### Enums
##### Camera Placements --- `E_CameraPlacements`
* -- None --
* Wheel, Angled
* Wheel, Overhead
* Wheel, Active Wedge
* Host
* Players (All)
* Player 1
* Player 2
* Player 3
* Puzzle Board
* DEBUG *(useful for, say, looking up close at the Pointer)*

##### Game UI Message Widgets --- `E_GameUIMessageWidgets`
Each value corresponds to a User Widget starting with UI_ and ending with the Enum display name (e.g. GameOver → UI_GameOver).

<dl>
	<dt>-- none --</dt>
	<dd>*(None selected)*</dd>
	<dt>PlayerNamesInput</dt>
	<dd>*Player input: Enter all 3 names*</dd>
	<dt>PlayerSpinBuySolve</dt>
	<dd>*Player input: Choose to Spin wheel, Buy vowel, or Solve puzzle*</dd>
	<dt>PlayerGuessConsonant</dt>
	<dd>*Player input: Guess a consonant (awarded wedge $ amount per correct letter)*</dd>
 	<dt>PlayerGuessVowel</dt>
	<dd>*Player input: Guess a vowel ($250 deducted after selection)*</dd>
 	<dt>PlayerAttemptSolve</dt>
	<dd>*Player input: Attempt to solve puzzle*</dd>
	<dt>IncorrectSolveGuess</dt>
	<dd>*Message: Incorrect solve guess, or user gave up on solve attempt (show message, end turn)*</dd>
 	<dt>RoundStart</dt>
	<dd>*Message: Round start (announce which round we're in)*</dd>
	<dt>RoundEnd</dt>
	<dd>*Message: Round end (declare correct guess, winner of round, prompt to continue)*</dd>
 	<dt>IncorrectLetterGuess</dt>
	<dd>*Message: Incorrect vowel or consonant guess (show message, end turn)*</dd>
	<dt>CorrectLetterGuess</dt>
	<dd>*Message: Correct vowel or consonant guess (highlight matching letter tile[s], reveal one at a time, repeat turn)*</dd>
 	<dt>FreeSpinSpace</dt>
	<dd>*Message: Landed on a Free Spin space (show graphic, add FS to player's inventory, repeat turn)*</dd>
	<dt>BankruptSpace</dt>
	<dd>*Message: Landed on a Bankrupt space (show graphic, current player loses all money, end turn*</dd>
	<dt>LoseTurnSpace</dt>
	<dd>*Message: Landed on a Lose Turn space (show graphic, end turn)*</dd>
	<dt>WildCardSpace</dt>
	<dd>*Message: Landed on a Wild Card space (TODO)*</dd>
	<dt>PrizeSpace</dt>
	<dd>*Message: Landed on a Prize space (TODO)*</dd>
	<dt>GameOver</dt>
	<dd>*Message: Game over! (show results, final total, prize(s), then let player choose Replay or Quit)*</dd>
</dl>

##### Letter Tile Characters --- `E_LetterTileCharacters`
This is a list of possible letters and other characters, with way more symbols than needed (just in case you might want to utilize them and include them in puzzles with their own Letter Tile textures). These values are used as Row Names in the datatables for Letter Tile character strings (`DT_LetterTileCharacterStrings`) and textures (`DT_LetterTileCharacterTextures`).

##### Player Animation Categories --- `E_PlayerAnimationCategories`
Useful for playing one random animation out of a categorized group.

* -- none --
* Greeting --- *(this is used when introducing each contestant)*
* Guessing
* Timed Out
* Defeated, Small
* Defeated, Big
* Celebrating, Small
* Celebrating, Big

##### Player Genders --- `E_PlayerGenders`

* -- none -- --- *(i.e. none selected)*
* Male
* Female
* Ambiguous

##### Puzzle Categories --- `E_PuzzleCategories`
These are all taken from the real world game show and have all appeared as puzzle categories at least once.

* -- none --
* Around the House
* Before and After
* Book Title
* Classic Movie
* Classic TV
* College Life
* Event
* Family
* Fictional Character(s)
* Fictional Place
* Food and Drink
* Fun and Games
* Headline
* In the Kitchen
* Landmark
* Living Thing
* Married Couple
* Megaword
* Movie Quote
* Movie Title
* Occupation
* On the Map
* People
* Person
* Phrase
* Place
* Proper Name
* Quotation
* Rhyme Time
* Rock On
* Same Letter
* Same Name
* Show Biz
* Slogan
* Song/Artist
* Song Lyrics
* Song Title
* Star and Role
* The 50's
* The 60's
* The 70's
* The 80's
* The 90's
* The Office --- *(bonus category!)*
* Thing
* Title/Author
* Title
* TV Show Title
* What Are You Doing?
* What Are You Wearing?
* Where Are We Going?

##### Space Action Types --- `E_SpaceActionTypes`
The *Space Action Types* Enum is a list of wheel space types, each of which can be handled separately (in the Game Mode's `Event Wheel Space Landed On` interface event). Among the following values, only *Money*, *Lose Turn*, and *Bankrupt* are handled by the default game, with the rest being planned for future updates. More can be added, then handled in the Game Mode blueprint --- don't forget to refresh the *Switch on E_SpaceActionTypes* node afterward, though, first.

The following list contains all of the wheel space types that are available to use.

<dl>
	<dt>Money</dt>
	<dd>Player is awarded `Money Value` dollars for each instance of a correct letter guess</dd>
	<dt>Prize</dt>
	<dd>**TODO:** Player is awarded a prize (randomized, collect all for achievement or something) on correct letter</dd>
	<dt>Lose Turn</dt>
	<dd>Player immediately loses their turn</dd>
	<dt>Bankrupt</dt>
	<dd>Player loses all of their money, then immediately loses their turn</dd>
	<dt>Free Vowel</dt>
	<dd>**TODO:** Player gets a free vowel guess (no money awarded, but no money taken either)</dd>
	<dt>Free Spin</dt>
	<dd>**TODO:** Player is awarded a Free Spin token that can be used only once at the end of any turn in order to have another turn to Solve or Spin. Player is then awarded `Money Value` dollars for each instance of a correct consonant guess (set to *-1* to skip consonant guess)</dd>
	<dt>Surprise</dt>
	<dd>**TODO:** Player is either: (a) awarded `Money Value` on the spot; or (b) shown a Bankrupt message. 50/50 random chance on landing to reveal which action is taken.</dd>
	<dt>Wild Card</dt>
	<dd>**TODO:** Player is awarded a Wild Card token that gives them an extra consonant selection in the Bonus Round, if they get there. Player is then awarded `Money Value` dollars for each instant of a correct consonant guess (set to -1 to skip consonant guess)</dd>
	<dt>Free Play</dt>
	<dd>**TODO:** Player is allowed to call any consonant OR a free vowel, with no penalty if the letter isn't in the puzzle. Player is awarded `Money Value` dollars for each correct consonant (not vowel) guess (set to -1 for ???)</dd>
	<dt>Jackpot</dt>
	<dd>**TODO:** At start of each Round, Jackpot Amount set to $5,000. With each spin that lands on a dollar amount, that amount is added to the Jackpot Amount (multiplied by the number of consonant occurrences). Upon landing on the Jackpot space, Player is awarded `Money Value` for each correct consonant (0 for no award). If Player then solves the puzzle correctly in the same turn, they win the Jackpot Amount! Otherwise, they lose their turn. Play then proceeds normally.</dd>
</dl>

#### Structs
##### Letter Tile Character String Structure --- `S_LetterTileCharacterString`
* **Character String** --- *String*
##### Letter Tile Character Texture Structure --- `S_LetterTileCharacterTexture`
* **Character Texture** --- *Texture*
##### Letter Tile Data Structure --- `S_LetterTileData`
1. **Content** --- *Letter Tile Characters Enum* (`E_LetterTileCharacters`)
	* Content of this tile (e.g. *Letter-A* for "A", *Period* for ".")
2. **Actor Ref** --- *Letter Tile* (`BP_LetterTile`) *Object Reference*
	* Reference to the actual Letter Tile actor BP spawned
3. **World Location** --- *Vector*
	* Position of Letter Tile actor blueprint in game world
4. **Row Position** --- *Integer*
	* Index of tile's position in row (0-11 or 0-13, depending on which row)
5. **Row Index** --- *Integer*
	* Index of row in which tile appears

##### Player Animations List Structure --- `S_PlayerAnimationsList`
* **Animations** --- *Array: Anim Montages*

##### Puzzle Answer Data Structure --- `S_PuzzleAnswerData`
1. **Category** --- *Puzzle Categories Enum* (`E_PuzzleCategories`)
2. **Answer** --- *String*
3. **Letter Count** --- *Integer*
	* Number of letters (A-Z only) in the Answer
4. **Is Bonus?** --- *Boolean*
	* Is this puzzle short enough to be a Bonus (final round) Puzzle?

##### Wheel Space Data Structure --- `S_WheelSpaceData`
1. **Space Text** --- *Text*
2. **Background Color** --- *Linear Color*
3. **Text Color** --- *Linear Color*
4. **Action Type** --- *Space Action Types Enum* (`E_SpaceActionTypes`)
5. **Money Value** --- *Integer*
6. **Text World Size** --- *Float*

#### Tables
##### Letter Tile Character Strings Table --- `DT_LetterTileCharacterStrings`
List of each allowed character's string.
* Row Structure: `S_LetterTileCharacterString`
* Number of Rows: **61**

##### Letter Tile Character Textures Table --- `DT_LetterTileCharacterTextures`
List of each allowed character's texture.
* Row Structure: `S_LetterTileCharacterTexture`
* Number of Rows: **57**

##### Puzzle Answers Table --- `DT_PuzzleAnswers`
List of all possible puzzle answers to choose from.
* Row Structure: `S_PuzzleAnswerData`
* Number of Rows: **79,006**

### User Interface
* **Player Message UI: Landed on Bankrupt Space** (`UI_BankruptSpace`) is displayed whenever the player lands on the Bankrupt space after spinning the wheel. The "Continue" button (a type of *Styled Button w/ Text* custom shared UI element) calls the Game Mode's `Close Message Widgets` interface event (with "End Turn?" set to *true*), allowing for the game to continue (after ending the current player's turn).
* **Player Message UI: Start of New Round** (`UI_BeginRoundMessage`) is displayed at the start of each round, displaying the current round number as stored in the GameMode blueprint for a several seconds before being removed by the same script responsible for its instantiation.
* **Player Message UI: Correct Letter Guessed** (`UI_CorrectLetterGuessMessage`) is displayed whenever the player guesses a correct letter (whether consonant or vowel), showing the letter itself and the number of occurrences of this letter in the puzzle answer.
* **Player Message UI: Player Wins Round** (`UI_EndRound`) is displayed at the end of a round once a player has solved the puzzle. The round number and winning contestant's name, amount won in the round, and amount won so far in the game are displayed. The "Continue" button calls the Game Mode's `Prep For Next Round` interface event, which removes this UI and continues gameplay with either the next round, or the Game Over Menu UI (if that was the last round).
* **Player Message UI: Game Over Menu** (`UI_GameOverMenu`) is displayed at the end of the game, after the Player Wins Round UI (`UI_EndRound`) for the final round is dismissed. The winning contestant's name and all final cash values are displayed. The "Start Over" button restarts with a fresh new game, while the "Quit Game" button quits the game.
* **Player Input UI: Guess a Consonant** (`UI_GuessConsonant`) is displayed after a player chooses to spin the wheel and it lands on a *Money*-type Wheel Wedge. The dollar amount that they landed on is displayed, as well as their player number and name. The buttons of any already-guessed consonants are disabled and the player must guess a letter by clicking one of the remaining enabled buttons.
* **Player Input UI: Guess a Vowel** (`UI_GuessVowel`) is displayed after a player chooses to buy a vowel. The buttons of any already-guessed vowels are disabled and the player must guess a letter by clicking the button representing one of the remaining enabled buttons.
* **Player Message UI: Incorrect Letter Guessed** (`UI_IncorrectLetterGuess`) is displayed after a player guesses a letter that is *not* present in the puzzle answer. The "Continue" button calls the Game Mode's `Close Message Widgets` interface event (with "End Turn?" set to *true*), allowing for the game to continue (after ending the current player's turn).
* **Player Message UI: Incorrect Solve Attempt** (`UI_IncorrectSolveAttempt`) is displayed after a player makes an incorrect attempt at solving the puzzle. The "Continue" button calls the Game Mode's `Close Message Widgets` interface event (with "End Turn?" set to *true*), allowing for the game to continue (after ending the current player's turn).
* **Player Message UI: Landed on Lose Turn Space** (`UI_LoseTurn`) is displayed after a player chooses to spin the wheel and it lands on a *Lose Turn*-type Wheel Wedge. The "Continue" button calls the Game Mode's `Close Message Widgets` interface event (with "End Turn?" set to *true*), allowing for the game to continue (after ending the current player's turn).
* **Player Text Input UI: Players Enter Names** (`UI_PlayerNamesInput`) is displayed at the start of every game, prompting the players to enter their names into the 3 available text input fields. The ON/OFF toggle determines whether or not we introduce the contestants before gameplay. This toggle is set to *On* by default. The "Start the Game!" button... well... starts the game. All three names are passed along to the Game Mode's `Set Player Names` and `Close Message Widgets` (with *End Turn?* set to *false*) interface events, in that order. After the player names are displayed on the Display HUD, we either introduce the contestants or go right into the first round of gameplay.
* **Player Input UI: Spin Wheel / Buy Vowel / Solve Puzzle** (`UI_PlayerSelectActionInput`) is displayed at the start of each player's turn. If the player has less than $250 (or whatever the Game Mode's `Vowel Cost` is set to), or there are No More Vowels left in the puzzle to uncover, the "Buy a Vowel" button is disabled. If there are No More Consonants left in the puzzle to uncover, the "Spin the Wheel" button is disabled. When either of the 3 buttons are clicked, the Game Mode's `Close Message Widgets` interface event is triggered. If the player clicks "Spin the Wheel", the Game Mode's `Spin Wheel` interface event is then triggered. If the player clicks "Buy a Vowel", the Game Mode's `Prompt Letter Guess` interface event is triggered, with Game Mode's `Vowel Cost` value passed as the event's *Ltr Value* argument. If the player clicks "Solve the Puzzle", the Game Mode's `Prompt Solve Puzzle` interface event is triggered.
* **Player Input UI: Solve Puzzle (Freeform Text)** (`UI_PlayerSolvePuzzleTextInput`) is displayed after the player chooses to solve the puzzle. Submitting an empty response or clicking on the "Cancel / Forfeit" button will trigger the Game Mode's `Show Incorrect Solve` interface event. Otherwise, the Game Mode's `Process Solve` interface event is triggered, with the entered text passed as the event's *Guess* argument, where the player's guess is then evaluated.

## Troubleshooting Common Issues
( List of frequent problems, e.g. wheel not spinning or puzzle board not rendering correctly, and solutions. )

## Frequently Asked Questions (FAQ)
( Answers to common questions about setup, customization, and extending the system. )

## Version History
Starting with most recent version first:

### Version 1.0
Initial release with full game system, and a simple three-round game loop.

## Support & Contact Information
For any questions regarding the game system, or any comments or suggestions for improvements or additions, please contact me at { FILL IN }, or submit a ticket at { FIND JIRA-LIKE FREE SITE }.

## Licensing & Legal
No trademarked or copyrighted terms, logos, sound effects, music, or other materials have been used in the Disc of Destiny project. Everything used within is either sourced from the public domain or Creative Commons (CC0), or created in-house, and verified to be free of any copyright or trademark violations. You are allowed to use anything and everything in this game system for your own legal purposes, including commercial use, under the Unreal Engine licensing terms and agreements. { FIND LINK }

### Credits
<dl>
	<dt>Main Menu music</dt>
	<dd>"Happy Happy Game Show" by Kevin MacLeod (incompetech.com)</dd>
	<dd>Licensed under Creative Commons: By Attribution 4.0 License (http://creativecommons.org/licenses/by/4.0/)</dd>
	<dt>Sound Effects</dt>
	<dd>All sound effects and stingers are either created by myself, or sourced from Freesound (Creative Commons 0 license)</dd>
</dl>
