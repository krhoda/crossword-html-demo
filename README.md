# Build a Crossword Puzzle with WASM Crossword Generator!

## Introduction

This tutorial will guide you through building a super basic crossword puzzle game in the browser using [WASM Crossword Generator](https://github.com/krhoda/wasm_crossword_generator). For maximum simplicity this guide will only use a single `index.html` page and build up all the logic and styling in tags. Everything here is fully usable in modern frameworks, and a slight more robust version of this approach is found [as a playable Create-React-App here](https://krhoda.github.io/anagram-crosswords) and [the repo for it is here](https://github.com/krhoda/anagram-crosswords). After finishing this guide, that project's code base is a decent place to look.

A big draw of [wasm_crossword_generator](https://github.com/krhoda/wasm_crossword_generator) is the portability compared to many other approaches to building WASM libraries. More info on that can be found in my reference [WASM NPM Library](https://github.com/krhoda/wasm_quicksort_example) which includes extensive examples of the library usage in things like Next.js, Node, and other major front-end frameworks. If you want to build a more complex crossword puzzle app, these sources should point you in the right direction.

## Table of Contents

[1. Initial Setup](#initial-setup)

[2. Generate the Puzzle](#generate-the-puzzle)

[3. Render the Blank Grid](#render-the-blank-grid)

[4. Build the Game Logic](#build-the-game-logic)

[5. Build the Game UI](#build-the-game-ui)

[6. Connect the Logic to the UI](#connect-the-logic-to-the-ui)

## Initial Setup

This guide should roughly follow the commits of this repo. To begin, create an empty directory. Then create an `index.html` file. In that file include the following contents:

```html
<!DOCTYPE html>
<html lang="en">

<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<meta http-equiv="X-UA-Compatible" content="ie=edge">
	<title>Crossword Puzzle</title>
	<link rel="icon" href="./favicon.ico" type="image/x-icon">
</head>

<body>
	<main>
		<h1>Crossword Puzzle</h1>
		<div id="game-container"></div>
	</main>
	<script src="https://cdn.jsdelivr.net/npm/wasm_crossword_generator@0.0.2/dist/umd/index.js"></script>
	<script defer>
		try {
			wasm_crossword_generator.CrosswordClient.initialize().then((client) => {
				window.CrosswordClient = client;
				console.log("All is well");
			})
		} catch (e) {
			console.error(e);
		}
	</script>
</body>

</html>
```

If you want that snazzy emoji game controller `favicon.ico`, go ahead and download it from this repo, otherwise, you'll get a broken image in the browser tab, but no other ill effects. From now on, only snippets of the file will be shown with `...` indicating unchanged areas.

The contents of the `head` tag are pretty bog-standard for the moment, the `body` tag has more interesting things going on. First, in the `main` tag is our header and a div with the id `game-container`. A couple steps later, we'll use that div to render the crossword puzzle's grid, and couple steps after that the player's controls.

After the `main` tag are a pair of script tags. The first imports `wasm_crossword_generator` from the `jsdelivr` CDN which mirrors NPM. The second `script` tag, set to `defer` so that it executes after the remote script is loaded, calls `initialize` on the `CrosswordClient`, then assigns the resulting `client` to the variable `window.CrosswordClient`. As we all know, it's not awesome to polute the `window` namespace, but it works perfectly well for this example.

If we run
```
$ npx http.server
```
in our directory, visit localhost:8080 and open the dev tools, we should see `All is well` printed to the dev console, indicating that the library loaded and initialized succesfully. You can also inspect `window.CrosswordClient` and see the methods it exposes. Now let's see them in action!

## Generate the Puzzle

We want to call `window.CrosswordClient`'s `generate_crossword_puzzle` fuction, so first thing we will do is create the `SolutionConf` structure to pass in as an arguement. The `SolutionConf`'s TypeScript definition and the definitions of the types that make up it's fields are shown here:
```typescript
export interface SolutionConf {
	words: Word[];
	max_words: number;
	width: number;
	height: number;
	requirements: CrosswordReqs | null;
	initial_placement: CrosswordInitialPlacement | null;
}

export interface Word {
	text: string;
	clue: string | null;
}

export interface CrosswordReqs {
	max_retries: number;
	min_letters_per_word: number | null;
	min_words: number | null;
	max_empty_columns: number | null;
	max_empty_rows: number | null;
}

export interface CrosswordInitialPlacement {
	min_letter_count: number | null;
	strategy: CrosswordInitialPlacementStrategy | null;
}

export type Direction = "Horizontal" | "Verticle";
export type CrosswordInitialPlacementStrategy = { Center: Direction } | // ... ;
```

Let's go through this field by field. At the top level, the `SolutionConf` contains four mandatory fields:

`words`: an array of `Word` objects, each `Word` contains a `text` field (which is what will be used to create the puzzle grid) and an optional `clue` field. For this example, we will be ignoring the `clue` field, but it's usage should be pretty straight forward. This array will be used to generate the grid by placing the `word.text`'s characters as squares in the grid.

`max_words`: an integer greater than 0, this is used to set an upper bound on crossword generation. If greater than `words.length`, it will be effectively unused.

`width` and `height`: represent the (maximum) width and height of the generated puzzle. This does not ensure that all rows and columns will be used. The `CrosswordReqs` object contains tools to make that adjustment.

The above entries are all that are strictly needed to attempt to generate a crossword, but in most applications, including this example, you will likely want some more configuration options which are available in the optional fields:

`requirements`: A(n optional) `CrosswordReqs` object, this will likely be used in most circumstances. The only mandatory field is `max_retries` which sets how many attempts the generator will make before giving up. This is because it is possible to pass a combination of `words` and `requirements` that have no valid ouput, so if `requirements` are supplied, the library also ensures that it will not loop forever. Execution is very fast, so setting this number anywhere from 200 to 1500 has been reasonable in anecdotal practice. `min_letters_per_word` allows for a lower bound on word size, if unsupplied/invalid, it will be set to three, because one and two letter words break grid generation. `min_words` sets a lower bound on the number of answers used in a crossword solution. `max_empty_columns`/`max_empty_rows` sets how many columns/rows (respectively) are permitted to be empty.

`initial_placement`: A(n optional) `CrosswordInitialPlacement` struct which allows for specifying how large the inital word will be and how it will be placed. Both fields are optional, `min_letter_count` sets a lower bound on the size of the first word placed, if unsupplied, defaults to `requirements.min_letters_per_word` or three if still unspecified. `strategy` allows the caller to specify where the first word will be placed, allowing for placement in the `Center`, one of the four corners, or a custom location, each variant having a configurable `Direction` (`"Horizontal"` or `"Verticle"`). If unsupplied, it will randomly pick from it's non-custom varients and a random direction.

So with all of that out of the way, let's write some code. We'll start with the `words` field by creating a constant that holds an array of `Word` objects. No shame in copying and pasting here!

```html
	<script defer>
		const words = [
			{
				"text": "friends",
				"clue": null
			},
			{
				"text": "dries",
				"clue": null
			},
			{
				"text": "end",
				"clue": null
			},
			{
				"text": "ends",
				"clue": null
			},
			{
				"text": "fed",
				"clue": null
			},
			{
				"text": "find",
				"clue": null
			},
			{
				"text": "finds",
				"clue": null
			},
			{
				"text": "fine",
				"clue": null
			},
			{
				"text": "finer",
				"clue": null
			},
			{
				"text": "fines",
				"clue": null
			},
			{
				"text": "fire",
				"clue": null
			},
			{
				"text": "fires",
				"clue": null
			},
			{
				"text": "friend",
				"clue": null
			},
			{
				"text": "red",
				"clue": null
			},
			{
				"text": "ride",
				"clue": null
			},
			{
				"text": "rides",
				"clue": null
			},
			{
				"text": "send",
				"clue": null
			},
			{
				"text": "side",
				"clue": null
			},
		];

		try {
		// ...
	</script>
```

In a real example, you would almost certainly have a data store that you would be pulling `Word` arrays from. The above example comes from the data source used [in this repo](https://github.com/krhoda/anagram-crosswords), which has a collection of array of `Word`s that it picks at random from.

Like that example, we will be making an "anagram crossword", meaning all of the words in the crossword are either an anagram of the largest word, or made from the letters of the largest word. This allows us to avoid writing clues -- the letters of largest word are all the clues a player needs.

With this `words` constant, we are now ready to define our config:
```html
	<script defer>
		const words = [
			// ...
		];

		const conf = {
			words,
			max_words: 100,
			width: 10,
			height: 10,
			requirements: {
				max_retries: 1500,
				min_letters_per_word: null,
				min_words: 12,
				max_empty_columns: 0,
				max_empty_rows: 0,
			},
			initial_placement: null,
		};
		try {
			//...
	</script>
```

Beyond setting `words`, this example sets `max_words` to greater than `words.length` allowing for the densest crosswords possible. `width`/`height` of ten means it is larger than the largest `Word.text.length` allowing all words to be used. In the `requirements`, we set `max_retries` to 1500 to maximize the chances of getting puzzle that meets our requirements. In other applications, you may use a lower amount of retries in favor of trying a new `Word`s array. To continue on the fields, we ignore `min_letters_per_word` because we're okay with the default of three, set `min_words` to twelve to get a relatively dense puzzle and set `max_empty_columns`/`max_empty_rows` to zero so that it looks square and filled out.

The last thing thing we need is to pick the "puzzle_type" of the puzzle. We will be using "PerWord", which means that the player guesses a word without guessing a location and the puzzle immediately informs them if the word is correct or not. "PlacedWord" is similar, but requires a location. "Classic" doesn't validate the player's guess but adds it to the grid. "PerWord" is the simplest to implement, so it's what we will see in action. First we will define the variables that will hold the `puzzle` and `puzzle_type`, then in the try block, we are ready to generate a puzzle:

```html
	<script defer>
		// ...
		let puzzle = null, puzzle_type = "PerWord"
		try {
			wasm_crossword_generator.CrosswordClient.initialize().then((client) => {
				window.CrosswordClient = client;
				let result = window.CrosswordClient.generate_crossword_puzzle(conf, puzzle_type);
				puzzle = result.puzzle;
				puzzle_type = result.puzzle_type;
				console.log(puzzle);
			})
		} catch (e) {
			console.error(e);
		}
	</script>
```

As you might notice, we are passed back the `puzzle_type` that we initally used. This variable will be used in subsuquent calls to the `CrosswordClient` so it knows how to process player guesses. We will also see the pattern of passing in a variable to a `CrosswordClient` method and re-defining that variable off of the result. This is because when memory is passed into WASM, it is dropped on the JS side and vice versa. The return value is value that was passed in by the JS after the WASM has mutated it. The approach of re-defining all variables off of calls using the variables (the classic functional programming style) is both the most optimized and least error prone way to interact with this library. Normally, destructuring would be used, but in a script tag, the intermediate variable of `result` will have to do.

The other half of the result, `puzzle`, is much more interesting. It has a TypeScript definition of:
```typescript
export interface Puzzle {
    solution: Solution;
    player_answers: PlacedWord[];
    grid: PuzzleRow[];
}
```
These fields are further defined as:

```typescript
export interface Placement {
    x: number;
    y: number;
    direction: Direction;
}

export interface PlacedWord {
    placement: Placement;
    word: Word;
}

export interface PuzzleSpace {
    char_slot: string | null;
    has_char_slot: boolean;
}

export interface PuzzleRow {
    row: PuzzleSpace[];
}

export interface Solution {
    grid: SolutionRow[];
    words: PlacedWord[];
    width: number;
    height: number;
}

export interface SolutionRow {
    row: (string | null)[];
}
```

The basic idea is that `solution` contains what the grid would look like if the `puzzle` were solved, as well as the list of `Word`s (or more percisely, `PlacedWord`s) that make up the answers. The `player_answers` field is a list of player submitted answers. In this example, it will only be the correct answers that the player has guessed, but in other play-modes, it may include bad answers. There are utility function to help with managing tracking of answer validity. Finally, the grid shows the crossword in a state of play, where player answer fill out the blanks in the puzzle. In this example, only correct guesses will populate the `puzzle.grid`. One thing worth noting, `SolutionRow.row`'s entries, if `string`s, will be of a single character, but there isn't a way to denote that in TypeScript.

With all of this in place, we are now ready to render the grid without displaying any answers.

## Render the Blank Grid

## Build the Game Logic

## Build the Game UI

## Connect the Logic to the UI
