<script>
    import {
        seed,
        getDate,
        yesterday,
        randomPastDate,
        loadWords,
        makeGenerate,
        makeCheck,
        makeSolve,
    } from "./litterboxed.js";

    import { fade } from "svelte/transition";
    import { onDestroy } from "svelte";
    import { onMount } from "svelte";

    onMount(async () => {
        let savedDate = localStorage.getItem("date") ?? getDate();
        await loadPuzzle(savedDate);
    });

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
    let easySolutionCount = 0;
    let totalSolutionCount = 0;
    let par = 0;
    let yesterdayLoaded = false;
    let showTodaySolutions = false;

    function calculatePar(easyCount, totalCount) {
        if (easyCount === 0 || (easyCount === 1 && totalCount === 1)) {
            return 6;
        } else if (easyCount === 1 && totalCount < 4) {
            return 5;
        } else if ((easyCount === 1 && totalCount >= 4) || easyCount === 2) {
            return 4;
        } else if (easyCount >= 3) {
            return 3;
        }
        return 6; // fallback safety
    }

    let loadPuzzle = async (dateStr) => {
        localStorage.setItem("date", dateStr);

        // load wordlists
        let easy = await loadWords("./easy.txt");
        let scrabble = await loadWords("./scrabble.txt");

        // generate puzzle letters
        generate = makeGenerate(easy);
        letters = generate(dateStr).join("").toUpperCase();
        localStorage.setItem("puzzle", letters);

        // set up solvers
        check = makeCheck(scrabble);
        solve = makeSolve(easy);
        solveAll = makeSolve(scrabble);

        // ✅ recalc counts
        easySolutionCount = solve(letters).length;
        totalSolutionCount = solveAll(letters).length;
        par = calculatePar(easySolutionCount, totalSolutionCount);

        // reset game state
        message = " ";
        previousWords = [];
        currentWord = "";
        displaySols = false;
        showTodaySolutions = false;
        yesterdayLoaded = dateStr === yesterday();
    };

    let loadTodayPuzzle = async () => {
        await loadPuzzle(getDate());
        yesterdayLoaded = false;
    };

    let loadYesterdayPuzzle = async () => {
        await loadPuzzle(yesterday());
        yesterdayLoaded = true;
    };

    let loadRandomPuzzle = async () => {
        await loadPuzzle(randomPastDate());
        yesterdayLoaded = false;
    };

    let loadTodaySolutions = () => {
        if (!showTodaySolutions) {
            let format = (s) => `${s[0]} - ${s[1]}`;
            solutions = solve(letters).map(format);
            allSolutions = solveAll(letters).map(format);
        }
        showTodaySolutions = !showTodaySolutions;
    };

    (async () => {
        if (localStorage.getItem("date") == getDate()) {
            let easy = await loadWords("./easy.txt");
            let scrabble = await loadWords("./scrabble.txt");
            generate = makeGenerate(easy);
            check = makeCheck(scrabble);
            solve = makeSolve(easy);
            solveAll = makeSolve(scrabble);
            message = " ";
        } else {
            loadTodayPuzzle();
        }
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
    let shakeAnimation = false;

    let enterWord = () => {
        if (done) return;
        if (currentWord.length < minlength) {
            triggerShake();
            return;
        }
        if (!check(currentWord)) {
            triggerShake();
            return;
        }

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

    function triggerShake() {
        shakeAnimation = false;

        // Use next tick / small timeout to re-add class
        setTimeout(() => {
            shakeAnimation = true;
        }, 10);
    }

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

    // GitHub repo URL
    const githubUrl = "https://github.com/steeb-k/alphabox";

    // Component state
    let isExpanded = true;
    let isHovering = false;
    let isVisible = true;

    // Close banner after delay
    onMount(() => {
        const timer = setTimeout(() => {
            if (!isHovering) {
                isExpanded = false;
            }
        }, 3000); // 3 seconds delay

        return () => clearTimeout(timer);
    });

    // Handle click - open GitHub in new tab
    function handleClick() {
        window.open(githubUrl, "_blank");
    }
</script>

<main>
    <h1>alphabox</h1>
    <p class="par">Try to get it in <strong>{par}</strong> or fewer words!</p>

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
                        <a href="https://github.com/steeb-k/alphabox"
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
            <span class="current-container">
                <span class:shake={shakeAnimation}>{currentWord}</span>
                <span class="caret">{caret}</span>
            </span>
        {:else}
            <!-- keeps layout reserved -->
            <span class="win-placeholder">YOU WIN!</span>

            <!-- animated overlay -->
            <span class="win-overlay" aria-hidden="true">
                {#each "YOU WIN!".split("") as char, i}
                    <span class="win-letter" style="--i:{i}">{char}</span>
                {/each}
            </span>
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
        <button on:click={enterWord}>Submit</button>
    </div>

    {#if !yesterdayLoaded}
        <p
            class="link"
            on:click={() => {
                loadYesterdayPuzzle();
            }}
        >
            Yesterday's Game
        </p>
    {:else}
        <p
            class="link"
            on:click={() => {
                loadTodayPuzzle();
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
    <br />
    <p
        class="link"
        on:click={() => {
            help = true;
        }}
    >
        Help
    </p>
    <div class="link" on:click={loadRandomPuzzle}>Random Game</div>

    <div
        class="github-banner"
        class:expanded={isExpanded}
        class:visible={isVisible}
        on:mouseenter={() => {
            isHovering = true;
            isExpanded = true;
        }}
        on:mouseleave={() => {
            isHovering = false;
            if (!isHovering) isExpanded = false;
        }}
        on:click={handleClick}
    >
        <div class="banner-content">
            <svg
                class="github-icon"
                xmlns="http://www.w3.org/2000/svg"
                viewBox="0 0 24 24"
            >
                <path
                    d="M12 0c-6.626 0-12 5.373-12 12 0 5.302 3.438 9.8 8.207 11.387.599.111.793-.261.793-.577v-2.234c-3.338.726-4.033-1.416-4.033-1.416-.546-1.387-1.333-1.756-1.333-1.756-1.089-.745.083-.729.083-.729 1.205.084 1.839 1.237 1.839 1.237 1.07 1.834 2.807 1.304 3.492.997.107-.775.418-1.305.762-1.604-2.665-.305-5.467-1.334-5.467-5.931 0-1.311.469-2.381 1.236-3.221-.124-.303-.535-1.524.117-3.176 0 0 1.008-.322 3.301 1.23.957-.266 1.983-.399 3.003-.404 1.02.005 2.047.138 3.006.404 2.291-1.552 3.297-1.23 3.297-1.23.653 1.653.242 2.874.118 3.176.77.84 1.235 1.911 1.235 3.221 0 4.609-2.807 5.624-5.479 5.921.43.372.823 1.102.823 2.222v3.293c0 .319.192.694.801.576 4.765-1.589 8.199-6.086 8.199-11.386 0-6.627-5.373-12-12-12z"
                />
            </svg>

            <div class="text-content">
                <span class="title">View on GitHub</span>
            </div>
        </div>
    </div>
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
        color: #ff3c3c;
        text-transform: uppercase;
        font-family: 'Squaresville', sans-serif;
        font-size: 7em;
        font-weight: 900;
        margin-top: 0;
        margin-bottom: -0.2em;

        /* Gradient from current color to transparent */
        background: linear-gradient(to bottom, currentColor, transparent);
        -webkit-background-clip: text;
        background-clip: text;
        -webkit-text-fill-color: transparent;
    }


    .par {
        margin-bottom: 2em;
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

    .current {
        position: relative;
        font-size: x-large;
        line-height: 1; /* baseline alignment */
        height: auto; /* let the line height handle spacing */
    }

    /* Placeholder occupies horizontal space only, zero height effect */
    .win-placeholder {
        visibility: hidden;
        display: inline-block;
        width: 100%; /* optional, match word width */
        height: 0; /* no extra vertical space */
    }

    /* Overlay: absolutely positioned horizontally, baseline-aligned vertically */
    .win-overlay {
        position: absolute;
        left: 50%; /* center horizontally */
        bottom: 0; /* align baseline with parent's text line */
        transform: translateX(-50%); /* horizontal centering */
        display: flex;
        align-items: baseline; /* keep letters on same line */
        gap: 0.05em; /* small space between letters */
        pointer-events: none;
        font-size: 2em;
        z-index: 10;
    }

    /* Letters bounce vertically only */
    .win-letter {
        display: inline-block;
        font-weight: 900;
        background: linear-gradient(
            90deg,
            #ff3e00,
            #ff0080,
            #ffeb00,
            #00ff99,
            #00c3ff,
            #7d00ff
        );
        -webkit-background-clip: text;
        background-clip: text;
        -webkit-text-fill-color: transparent;

        /* Restore outline */
        -webkit-text-stroke: 2px var(--text-color);
        text-stroke: 2px var(--text-color); /* fallback for future */

        animation: win-bounce 0.7s ease infinite;
        animation-delay: calc(var(--i) * 0.07s);
        will-change: transform;
    }

    /* Bounce: vertical movement only */
    @keyframes win-bounce {
        0% {
            transform: translateY(0);
        }
        40% {
            transform: translateY(-8px);
        }
        70% {
            transform: translateY(2px);
        }
        100% {
            transform: translateY(0);
        }
    }

    @media (prefers-reduced-motion: reduce) {
        .win-letter {
            animation: none;
        }
    }

    @keyframes shake-red {
        0% {
            transform: translateX(0);
            color: var(--text-color);
        }
        25% {
            transform: translateX(-4px);
            color: red;
        }
        50% {
            transform: translateX(4px);
            color: red;
        }
        75% {
            transform: translateX(-4px);
            color: red;
        }
        100% {
            transform: translateX(0);
            color: var(--text-color);
        }
    }

    .shake {
        display: inline; /* ensure no extra box */
        animation: shake-red 0.2s ease;
    }
    .current-container {
        position: relative;
        display: inline-block; /* keeps width to the word */
    }

    .current-container > .caret {
        position: absolute;
        left: 100%; /* start right after the word span */
        margin-left: 0.1em; /* small gap between last letter and caret */
        top: 50%;
        transform: translateY(-50%);
        font-size: inherit;
        line-height: 1;
    }

    .current-container > span {
        display: inline-block;
        line-height: 1; /* matches caret positioning */
    }

    .github-banner {
        position: fixed;
        top: 20px;
        right: 20px;
        background: var(--button-bg-color);
        color: var(--text-color);
        border: 2px solid var(--button-text-color);
        border-radius: 30px;
        padding: 10px 20px;
        cursor: pointer;
        box-shadow: 0px 0px 12px var(--text-color);
        transition: all 0.3s ease;
        z-index: 1000;
        display: flex;
        align-items: center;
        justify-content: center; /* Center content */
        overflow: hidden;
        width: auto;
        max-width: 200px;
        height: 40px;
        box-sizing: border-box; /* This is the key fix! */
    }

    .github-banner:not(.expanded) {
        max-width: 40px;
        padding: 10px; /* Equal padding all around */
    }

    .banner-content {
        display: flex;
        align-items: center;
        gap: 10px;
    }

    /* Add this rule to fix the collapsed state */
    .github-banner:not(.expanded) .banner-content {
        gap: 0; /* Remove gap when collapsed */
        justify-content: center; /* Center the icon */
    }

    .github-icon {
        width: 24px;
        height: 24px;
        fill: var(--text-color);
        flex-shrink: 0;
    }

    .text-content {
        white-space: nowrap;
        overflow: hidden;
        transition: opacity 0.2s ease;
        font-weight: 600;
        font-size: 14px;
    }

    .github-banner:not(.expanded) .text-content {
        opacity: 0;
        width: 0;
        margin: 0; /* Remove margin when collapsed */
    }

    @font-face {
    font-family: 'Squaresville';
    src: url('/SQUARESV.TTF') format('truetype');
    font-weight: normal;
    font-style: normal;
    font-display: swap; /* Optional: improves performance */
}
</style>
