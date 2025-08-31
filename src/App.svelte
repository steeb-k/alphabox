<script>
    import {
        seed,
        getDate,
        yesterday,
        loadWords,
        makeGenerate,
        makeCheck,
        makeSolve,
    } from "./litterboxed.js";

    import { fade } from "svelte/transition";
    import { onDestroy } from "svelte";

    onDestroy(() => {
        if (animationInterval) cancelAnimationFrame(animationInterval);
    });

    let message = "loading";
    let alert = (msg) => {
        message = msg;
        setTimeout(() => {
            message = "";
        }, 1000);
    };

    let minlength = 3;
    let side = 30;
    let x = 6;
    let y = 6;
    let circles = [];
    let letters_pos = [];
    let letter_offset = 3.5;
    let letter_size = 4;
    let letters = "";
    if (localStorage.getItem("date") != getDate()) {
        localStorage.removeItem("puzzle");
    }
    letters = localStorage.getItem("puzzle") || "            ";
    let generate, check, solve, solveAll;
    let displaySols = false;
    let solutions, allSolutions;
    let yesterdayLoaded = false; // New variable
    let showTodaySolutions = false;
    let loadTodaySolutions = () => {
        if (!showTodaySolutions) {
            let format = (s) => `${s[0]} - ${s[1]}`;
            solutions = solve(letters).map(format);
            allSolutions = solveAll(letters).map(format);
        }
        showTodaySolutions = !showTodaySolutions;
    };
    // New function to load yesterday's solutionslet showTodaySolutions = false;
    let loadYesterdaySolutions = () => {
        let yesterdayPuzzle = generate(yesterday()).join("").toUpperCase();
        letters = yesterdayPuzzle;
        previousWords = [];
        currentWord = "";
        let format = (s) => `${s[0]} - ${s[1]}`;
        solutions = solve(yesterdayPuzzle).map(format);
        allSolutions = solveAll(yesterdayPuzzle).map(format);
        displaySols = true; // This will now be triggered by the new button
    };
    (async () => {
        let y = window.location.hash == "#y";
        seed(getDate());
        let easy = await loadWords("./easy.txt");
        generate = makeGenerate(easy);
        letters = generate(getDate()).join("").toUpperCase();
        localStorage.setItem("puzzle", letters);
        localStorage.setItem("date", getDate());
        message = "loading dict";
        let scrabble = await loadWords("./scrabble.txt");
        check = makeCheck(scrabble);
        message = " ";
        solve = makeSolve(easy);
        solveAll = makeSolve(scrabble);
    })();

    let hitboxes = [];
    for (let i = 0; i < 3; i++) {
        let offset = (side / 3) * (i + 0.5);
        circles.push({ x: x, y: y + offset });
        letters_pos.push({ x: x - letter_offset, y: y + offset });
        hitboxes.push({
            x: x - letter_offset - letter_size / 2,
            y: y + offset - letter_size / 2,
            width: letter_size + letter_offset,
            height: letter_size,
        });

        circles.push({ x: x + side, y: y + offset });
        letters_pos.push({ x: x + side + letter_offset, y: y + offset });
        hitboxes.push({
            x: x + side - letter_size / 2,
            y: y + offset - letter_size / 2,
            width: letter_size + letter_offset,
            height: letter_size,
        });

        circles.push({ x: x + offset, y: y });
        letters_pos.push({ x: x + offset, y: y - letter_offset });
        hitboxes.push({
            x: x + offset - letter_size / 2,
            y: y - letter_offset - letter_size / 2,
            width: letter_size,
            height: letter_size + letter_offset,
        });

        circles.push({ x: x + offset, y: y + side });
        letters_pos.push({ x: x + offset, y: y + side + letter_offset });
        hitboxes.push({
            x: x + offset - letter_size / 2,
            y: y + side - letter_size / 2,
            width: letter_size,
            height: letter_size + letter_offset,
        });
    }
    let stroke = 0.3;
    let index = (i) => (i % 4) * 3 + Math.floor(i / 4);
    let revindex = (i) => (i % 3) * 4 + Math.floor(i / 3);
    let currentWord = "";
    $: lastLetter = currentWord ? letters.indexOf(currentWord.slice(-1)) : -1;
    let selectLetter = (i) => {
        if (done) return;
        if (Math.floor(lastLetter / 3) != Math.floor(i / 3))
            currentWord = currentWord + letters[i];
    };
    let deleteLetter = () => {
        currentWord = currentWord.slice(0, -1);
        if (currentWord == "")
            if (previousWords.length) {
                currentWord = previousWords.pop();
                previousWords = previousWords;
            }
    };

    let previousWords = [];
    $: done = [...Array(12).keys()].every(
        (i) => previousWords.join("").indexOf(letters[i]) > -1,
    );
    let enterWord = () => {
        if (done) return;
        if (currentWord.length < minlength) return alert("Too short");
        if (!check(currentWord)) return alert("Not a word");

        // Start animation
        if (currentWord.length > 1) {
            animatingWord = currentWord;
            totalPathLength = calculatePathLength(currentWord);
            currentPathLength = totalPathLength; // Start with full offset

            // Clear any existing animation
            if (animationInterval) cancelAnimationFrame(animationInterval);

            const startTime = Date.now();

            const animate = () => {
                const elapsed = Date.now() - startTime;
                const progress = Math.min(elapsed / animationDuration, 1);

                currentPathLength = totalPathLength * (1 - progress);

                if (progress < 1) {
                    animationInterval = requestAnimationFrame(animate);
                } else {
                    // After animation completes, add to previous words
                    previousWords = [...previousWords, animatingWord];
                    currentWord = animatingWord.slice(-1);
                    animatingWord = null;
                    if (done) currentWord = "";
                }
            };

            animationInterval = requestAnimationFrame(animate);
        } else {
            // For single letter words
            previousWords = [...previousWords, currentWord];
            currentWord = currentWord.slice(-1);
            if (done) currentWord = "";
        }
    };
    let caret = "|";
    setInterval(() => {
        caret = caret ? "" : "|";
    }, 500);
    $: words = previousWords.join(" - ");

    // UPDATED letterColor function
    let letterColor = (i) => {
        if (i == lastLetter) return "#ff3e00"; // Accent color for last letter

        // Check if this letter is used in previous words (not current word)
        const usedInPreviousWords =
            previousWords.join("").indexOf(letters[i]) > -1;
        const usedInCurrentWord = currentWord.indexOf(letters[i]) > -1;

        if (usedInPreviousWords && !usedInCurrentWord) return "var(--bg-color)"; // Grey for letters used in previous words only

        // For currently selected letters in current word (excluding last letter)
        if (usedInCurrentWord && i !== lastLetter) {
            return "var(--text-color)"; // Make circle fill transparent, text color handles the letter
        }

        return "var(--text-color)"; // Default circle fill
    };

    // Animation variables
    let animatingWord = null;
    let animationProgress = 0;
    let animationDuration = 800; // milliseconds per word
    let animationInterval;
    let totalPathLength = 0;
    let currentPathLength = 0;

    // Function to create SVG path data for the animated word
    const getAnimatedPath = (word) => {
        if (!word || word.length < 2) return "";

        let pathData = "";
        for (let i = 0; i < word.length; i++) {
            const circle = circles[revindex(letters.indexOf(word[i]))];
            if (i === 0) {
                pathData = `M ${circle.x} ${circle.y} `;
            } else {
                pathData += `L ${circle.x} ${circle.y} `;
            }
        }
        return pathData;
    };

    const calculatePathLength = (word) => {
        let length = 0;
        for (let i = 0; i < word.length - 1; i++) {
            const start = circles[revindex(letters.indexOf(word[i]))];
            const end = circles[revindex(letters.indexOf(word[i + 1]))];
            const dx = end.x - start.x;
            const dy = end.y - start.y;
            length += Math.sqrt(dx * dx + dy * dy);
        }
        return length;
    };
    document.addEventListener("keydown", function (event) {
        if (event.key == "Enter") {
            enterWord();
        } else if (event.key == "Backspace") {
            deleteLetter();
        } else {
            let i = letters.indexOf(event.key.toUpperCase());
            if (i != -1) selectLetter(i);
        }
    });
    let help = false;
    let modal;
</script>

<main>
    <h1>litter boxed</h1>

    {#if help}
        <div
            class="modal"
            bind:this={modal}
            on:click={(evt) => {
                help = evt.target != modal;
            }}
        >
            <div class="modal-content">
                <h3 style="margin-top: .5em">How to Play</h3>
                <ul>
                    <li>Connect letters to spell words</li>
                    <li>Words must be at least 3 letters long</li>
                    <li>Letters can be reused</li>
                    <li>Consecutive letters cannot be from the same side</li>
                    <li>
                        The last letter of a word becomes the first letter of
                        the next word <br />
                        e.g. THY > YES > SINCE
                    </li>
                    <li>Words cannot be proper nouns or hyphenated</li>
                    <li>Use all letters to solve!</li>
                    <li>
                        There is always a solution with two words that repeats
                        only the common letter <br />
                        e.g. LANDS > SECURITY
                    </li>
                    <p>
                        Inspired by the <a
                            href="https://www.nytimes.com/puzzles/letter-boxed"
                            >NYT's Letter Boxed</a
                        >.
                    </p>
                    <p>
                        <a href="https://github.com/louisabraham/litterboxed"
                            >Source code and trivia</a
                        >
                    </p>
                </ul>
            </div>
        </div>
    {/if}
    <div class="current">
        {#if message}
            <span
                style="position: absolute; left: 0; right: 0; font-size: medium;"
                transition:fade
                class="message">{message}</span
            >
        {/if}
        {#if !done}
            <span style="display: inline">{currentWord}</span><span
                class="unselectable"
                style={(currentWord ? "width: 0em;" : "") +
                    "display: inline-block;"}>{caret}</span
            >
        {:else}
            <span class="blink" style="display: inline">YOU WIN!</span>
        {/if}
        <hr
            style="min-width: 10em; max-width: 13em;
            border:1px solid var(--svg-stroke-color); margin-top: 0"
        />
    </div>
    <div class="words">
        <p
            style="width: fit-content; margin: auto"
            on:click={() => {
                navigator.clipboard.writeText(words), alert("copied");
            }}
        >
            {words}
        </p>
    </div>
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 42 42">
        <rect
            {x}
            {y}
            width={side}
            height={side}
            stroke="var(--svg-stroke-color)"
            stroke-width={stroke}
            fill="none"
        />
        {#each [...previousWords, currentWord] as word, pos}
            {#each word as l, i}
                {#if i + 1 < word.length}
                    <line
                        x1={circles[revindex(letters.indexOf(l))].x}
                        y1={circles[revindex(letters.indexOf(l))].y}
                        x2={circles[revindex(letters.indexOf(word[i + 1]))].x}
                        y2={circles[revindex(letters.indexOf(word[i + 1]))].y}
                        stroke={pos == previousWords.length
                            ? "#ff3e00"
                            : "#ff3e0080"}
                        stroke-width={stroke}
                        stroke-dasharray={pos == previousWords.length ? 2 : 0}
                    />
                {/if}
            {/each}
        {/each}

        {#if !animatingWord && currentWord}
            {#each currentWord as l, i}
                {#if i + 1 < currentWord.length}
                    <line
                        x1={circles[revindex(letters.indexOf(l))].x}
                        y1={circles[revindex(letters.indexOf(l))].y}
                        x2={circles[
                            revindex(letters.indexOf(currentWord[i + 1]))
                        ].x}
                        y2={circles[
                            revindex(letters.indexOf(currentWord[i + 1]))
                        ].y}
                        stroke="#ff3e00"
                        stroke-width={stroke}
                        stroke-dasharray="2"
                    />
                {/if}
            {/each}
        {/if}
        {#if animatingWord}
            <path
                d={getAnimatedPath(animatingWord)}
                stroke="#ff3e00"
                stroke-width={stroke}
                fill="none"
                stroke-dasharray={totalPathLength}
                stroke-dashoffset={currentPathLength}
            />
        {/if}
        {#each circles as c, i}
            <circle
                cx={c.x}
                cy={c.y}
                r="1"
                fill={(lastLetter, currentWord, letterColor(index(i)))}
                stroke="var(--svg-stroke-color)"
                stroke-width={stroke}
            />
            <text
                text-anchor="middle"
                dominant-baseline="central"
                x={letters_pos[i].x}
                y={letters_pos[i].y}
                font-size={letter_size}
                fill="var(--text-color)"
            >
                {letters[index(i)]}
            </text>
            <rect
                x={hitboxes[i].x}
                y={hitboxes[i].y}
                width={hitboxes[i].width}
                height={hitboxes[i].height}
                fill="none"
                stroke="none"
                stroke-width=".1"
                pointer-events="fill"
                on:click={() => selectLetter(index(i))}
            />
        {/each}
    </svg>

    <div class="buttons">
        <button on:click={deleteLetter}>Delete</button>
        <button on:click={enterWord}>Enter</button>
    </div>

    {#if !yesterdayLoaded}
        <p
            class="link"
            on:click={() => {
                let yesterdayPuzzle = generate(yesterday())
                    .join("")
                    .toUpperCase();
                letters = yesterdayPuzzle;
                previousWords = [];
                currentWord = "";
                yesterdayLoaded = true;
                showTodaySolutions = false;
                displaySols = false; // Reset solutions display when switching to yesterday
            }}
        >
            Yesterday's Game
        </p>
    {:else}
        <p
            class="link"
            on:click={() => {
                if (localStorage.getItem("date") != getDate()) {
                    localStorage.removeItem("puzzle");
                }
                letters = localStorage.getItem("puzzle") || "            ";
                generate(getDate());
                letters = generate(getDate()).join("").toUpperCase();
                localStorage.setItem("puzzle", letters);
                localStorage.setItem("date", getDate());
                previousWords = [];
                currentWord = "";
                yesterdayLoaded = false;
                showTodaySolutions = false; // Reset today's solutions when switching back
            }}
        >
            Today's Game
        </p>
    {/if}

    {#if !yesterdayLoaded}
        <p class="link" on:click={loadTodaySolutions}>
            {#if showTodaySolutions}
                Hide Solutions
            {:else}
                View Solutions
            {/if}
        </p>
    {/if}

    {#if showTodaySolutions && !yesterdayLoaded}
        <div class="solutions">
            <p>Some solutions for today</p>
            {#each solutions as sol}
                <p style="font-weight: bold">{sol}</p>
            {/each}
            {#each allSolutions as sol}
                {#if !solutions.includes(sol)}
                    <p>{sol}</p>
                {/if}
            {/each}
        </div>
    {/if}

    {#if yesterdayLoaded}
        <p
            class="link"
            on:click={() => {
                if (!displaySols) {
                    let yesterdayPuzzle = generate(yesterday())
                        .join("")
                        .toUpperCase();
                    let format = (s) => `${s[0]} - ${s[1]}`;
                    solutions = solve(yesterdayPuzzle).map(format);
                    allSolutions = solveAll(yesterdayPuzzle).map(format);
                }
                displaySols = !displaySols;
            }}
        >
            {#if displaySols}
                Hide Solutions
            {:else}
                View Solutions
            {/if}
        </p>
    {/if}

    {#if displaySols && yesterdayLoaded}
        <div class="solutions">
            <p>Some solutions for yesterday</p>
            {#each solutions as sol}
                <p style="font-weight: bold">{sol}</p>
            {/each}
            {#each allSolutions as sol}
                {#if !solutions.includes(sol)}
                    <p>{sol}</p>
                {/if}
            {/each}
        </div>
    {/if}
    <br>
    <p
        class="link"
        on:click={() => {
            help = true;
        }}
    >
        Help
    </p>
</main>

<style global>
    /* Important: Add this global style import to ensure global.css is bundled */
    @import "./global.css";

    main {
        text-align: center;
        padding: 1em;
        margin: 0 auto;
    }

    h1 {
        color: #d10000;
        text-transform: uppercase;
        font-size: 3em;
        font-weight: 900;
        margin-top: 0;
        margin-bottom: 0.4em;
    }

    .unselectable {
        -webkit-touch-callout: none;
        -webkit-user-select: none;
        -khtml-user-select: none;
        -moz-user-select: none;
        -ms-user-select: none;
        user-select: none;
    }
    .current {
        font-size: x-large;
        position: static;
    }

    .words {
        height: 1em;
        font-size: large;
    }
    .buttons > button {
        width: 100px;
        text-align: center;
    }

    svg {
        width: 100%;
        max-width: 400px;
        margin-top: 1em;
        margin-bottom: 1em;
    }

    .message {
        margin-top: -1.4em;
    }

    .blink {
        animation: blinker 1s infinite;
    }

    .link {
        display: inline;
        margin: 0.5em;
        color: var(--link-color);
        cursor: pointer;
    }

    .link:hover {
        text-decoration: underline;
    }

    .modal {
        position: fixed;
        z-index: 1;
        left: 0;
        top: 0;
        width: 100%;
        height: 100%;
        overflow: auto;
        background-color: rgba(0, 0, 0, 0.4);
    }

    .modal-content {
        background-color: var(--modal-bg-color);
        margin: 6em auto;
        padding: 10px;
        border: 1px solid var(--border-color);
        border-radius: 10px;
        max-width: 400px;
    }
    .modal-content > ul {
        text-align: left;
    }
    .line-animation {
        transition: stroke-dashoffset 0.75s linear;
    }

    @keyframes blinker {
        50% {
            opacity: 0;
        }
    }

    @media (max-width: 350px) {
        h1 {
            font-size: 2.5em;
        }
        .modal-content {
            margin: 5.5em auto;
        }
        .message {
            margin-top: -1.2em;
        }
    }
</style>
