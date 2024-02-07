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

If you want that snazzy emoji game controller `favicon.ico`, go ahead and download it from this repo, otherwise, you'll get a broken image. From now on, only snippets of the file will be shown with `...` indicating unchanged areas.

The contents of the `head` tag are pretty bog-standard for the moment, the `body` tag has more interesting things going on. First, in the `main` tag is our header and a div with the id `game-container`. A couple steps later, we'll use that div to render the crossword puzzle's grid, and couple steps after that the player's controls. After the `main` tag are a pair of script tags. The first imports `wasm_crossword_generator` from the `jsdelivr` CDN which mirrors NPM. The second `script` tag, set to `defer` so that it executes after the remote script is loaded, calls `initialize` on the `CrosswordClient`, then assigns the resulting `client` to the variable `window.CrosswordClient`. As we all know, it's not awesome to polute the `window` name space, but it works perfectly well for this example.

If we run
```
$ npx http.server
```
in our directory, visit localhost:8080 and open the dev tools, we should see `All is well` printed to the dev console, indicating that the library loaded and initialized succesfully. You can also inspect `window.CrosswordClient` and see the methods it exposes. Now let's see them in action!

## Generate the Puzzle

## Render the Blank Grid

## Build the Game Logic

## Build the Game UI

## Connect the Logic to the UI
