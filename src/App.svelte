<script>
    import { onDestroy, onMount } from "svelte";
    import { fade, fly } from "svelte/transition";
    import { cubicIn, cubicOut } from "svelte/easing";
    import html2canvas from "html2canvas";
    import {
        getDate,
        yesterday,
        randomPastDate,
        loadWords,
        makeGenerate,
        makeCheck,
        makeSolve,
        getStreakData,
        updateStreak,
        shouldShowHelp,
    } from "./litterboxed.js";
    import "./global.css";

    // ===== VARIABLE DECLARATIONS =====
    // Game state variables
    let message = "loading";
    let currentWord = "";
    let previousWords = [];
    let done = false;
    let shakeAnimation = false;
    let displaySols = false;
    let solutions, allSolutions;
    let easySolutionCount = 0;
    let totalSolutionCount = 0;
    let par = 0;
    let yesterdayLoaded = false;
    let showTodaySolutions = false;

    // UI state variables
    let help = false;
    let solutionsModal = false;
    let showMenu = false;
    let modal;
    let isExpanded = true;
    let isHovering = false;
    let isVisible = true;
    let isSidebarHovered = false;

    // Puzzle configuration
    let minlength = 3;
    let side = 30;
    let x = 6;
    let y = 6;
    let stroke = 0.3;
    let letter_offset = 3.5;
    let letter_size = 4;
    let letters = localStorage.getItem("puzzle") || "            ";
    let url = "steeb-k.github.io/alphabox";

    // Streak tracking
    let streakData = null;
    let showStreakModal = false;
    let showHelpOnLoad = false;

    // Animation variables
    let animatingWord = null;
    let animationDuration = 800;
    let animationInterval;
    let totalPathLength = 0;
    let currentPathLength = 0;

    // DOM references
    let screenshotArea;
    let currentEl;
    let scaleFactor = 1;

    // Game functions (will be assigned)
    let generate, check, solve, solveAll;

    // Constants and derived data
    let githubUrl = "https://github.com/steeb-k/alphabox";
    let caret = "|";
    let circles = [];
    let letters_pos = [];
    let hitboxes = [];
    let sidebarTimeout;

    // ===== COMPUTED PROPERTIES =====
    $: lastLetter = currentWord ? letters.indexOf(currentWord.slice(-1)) : -1;
    $: done = [...Array(12).keys()].every(
        (i) => previousWords.join("").indexOf(letters[i]) > -1,
    );
    $: words = previousWords.join(" - ");
    $: currentText = [...previousWords, currentWord]
        .filter(Boolean)
        .join(" - ");

    $: if (currentEl && currentText) {
        requestAnimationFrame(() => {
            requestAnimationFrame(() => {
                const parent = currentEl.parentElement;
                const parentWidth = parent.offsetWidth;
                const contentWidth = currentEl.scrollWidth;

                if (contentWidth > parentWidth) {
                    scaleFactor = Math.max(0.3, parentWidth / contentWidth);
                    currentEl.style.transform = `scale(${scaleFactor})`;
                } else {
                    scaleFactor = 1;
                    currentEl.style.transform = "scale(1)";
                }
            });
        });
    }

    // ===== LIFECYCLE FUNCTIONS =====
    onMount(async () => {
        applyThemePreference();
        let savedDate = localStorage.getItem("date") ?? getDate();
        await loadPuzzle(savedDate);

        // Check streak data after loading puzzle
        streakData = getStreakData();
        showHelpOnLoad = shouldShowHelp();

        // Show appropriate modal
        if (showHelpOnLoad) {
            help = true;
        } else {
            // Always show streak modal for returning users
            showStreakModal = true;
        }

        // Close banner after delay
        const timer = setTimeout(() => {
            if (!isHovering) {
                isExpanded = false;
            }
        }, 3000);

        return () => clearTimeout(timer);
    });

    onDestroy(() => {
        if (animationInterval) cancelAnimationFrame(animationInterval);
    });

    // ===== GAME LOGIC FUNCTIONS =====
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
        return 6;
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

        // recalc counts
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

    // ===== UI INTERACTION FUNCTIONS =====
    let index = (i) => (i % 4) * 3 + Math.floor(i / 4);
    let revindex = (i) => (i % 3) * 4 + Math.floor(i / 3);

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

        if (currentWord.length > 1) {
            animatingWord = currentWord;
            totalPathLength = calculatePathLength(currentWord);
            currentPathLength = totalPathLength;
            if (animationInterval) cancelAnimationFrame(animationInterval);
            const startTime = Date.now();

            // Animation function
            const animate = () => {
                const elapsed = Date.now() - startTime;
                const progress = Math.min(elapsed / animationDuration, 1);
                currentPathLength = totalPathLength * (1 - progress);
                if (progress < 1) {
                    animationInterval = requestAnimationFrame(animate);
                } else {
                    previousWords = [...previousWords, animatingWord];
                    currentWord = animatingWord.slice(-1);
                    animatingWord = null;

                    // Check if game is won and update streak
                    if (done) {
                        handleGameCompletion();
                    }
                }
            };
            animationInterval = requestAnimationFrame(animate);
        } else {
            previousWords = [...previousWords, currentWord];
            currentWord = currentWord.slice(-1);

            // Check if game is won and update streak
            if (done) {
                handleGameCompletion();
            }
        }
    };

    function triggerShake() {
        shakeAnimation = false;
        setTimeout(() => {
            shakeAnimation = true;
        }, 10);
    }

    // ===== ANIMATION FUNCTIONS =====
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

    function handleGameCompletion() {
        if (!done) return;

        // Only update streak and show modal for today's game
        const isTodaysGame = localStorage.getItem("date") === getDate();

        if (isTodaysGame) {
            // Update streak only for today's game
            const updatedStreak = updateStreak(true);
            console.log("Today's game won! Streak updated:", updatedStreak);

            // Update the UI
            streakData = updatedStreak;

            // Show streak modal AFTER the UI has updated
            setTimeout(() => {
                showStreakModal = true;
            }, 100); // Short delay to ensure UI updates first
        }
    }

    // Add this reactive statement to watch for game completion
    $: if (done) {
        handleGameCompletion();
    }

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

    // ===== UI HELPER FUNCTIONS =====
    let letterColor = (i) => {
        if (i == lastLetter) return "#ff3e00";

        const usedInPreviousWords =
            previousWords.join("").indexOf(letters[i]) > -1;
        const usedInCurrentWord = currentWord.indexOf(letters[i]) > -1;

        if (usedInPreviousWords && !usedInCurrentWord) return "var(--bg-color)";

        if (usedInCurrentWord && i !== lastLetter) {
            return "var(--text-color)";
        }

        return "var(--text-color)";
    };

    function handleEscapeKey(event) {
        if (event.key === "Escape") {
            if (help) {
                help = false;
                event.preventDefault();
            } else if (solutionsModal) {
                solutionsModal = false;
                event.preventDefault();
            } else if (showMenu) {
                showMenu = false;
                event.preventDefault();
            } else if (showStreakModal) {
                showStreakModal = false;
                event.preventDefault();
            }
        }
    }

    function handleClick() {
        window.open(githubUrl, "_blank");
    }

    function openSidebarOnHover() {
        if (!showMenu) {
            clearTimeout(sidebarTimeout);
            sidebarTimeout = setTimeout(() => {
                showMenu = true;
            }, 300);
        }
    }

    function closeSidebarOnLeave() {
        if (showMenu && !isSidebarHovered) {
            clearTimeout(sidebarTimeout);
            sidebarTimeout = setTimeout(() => {
                showMenu = false;
            }, 500);
        }
    }

    function setSidebarHover(state) {
        isSidebarHovered = state;
        if (state) {
            clearTimeout(sidebarTimeout);
        } else {
            closeSidebarOnLeave();
        }
    }

    function showCustomAlert(message) {
        const alertBox = document.createElement("div");
        alertBox.textContent = message;

        alertBox.style.position = "fixed";
        alertBox.style.top = "50%";
        alertBox.style.left = "50%";
        alertBox.style.transform = "translate(-50%, -50%)";
        alertBox.style.zIndex = "9999";
        alertBox.style.backgroundColor = "var(--modal-bg-color)";
        alertBox.style.color = "var(--modal-text-color)";
        alertBox.style.padding = "1em 2em";
        alertBox.style.borderRadius = "10px";
        alertBox.style.fontSize = "1.5em";
        alertBox.style.fontWeight = "bold";
        alertBox.style.textAlign = "center";
        alertBox.style.transition = "opacity 0.5s ease";
        alertBox.style.opacity = "1";

        document.body.appendChild(alertBox);

        setTimeout(() => {
            alertBox.style.opacity = "0";
            setTimeout(() => {
                alertBox.remove();
            }, 500);
        }, 1500);
    }

    // ===== SCREENSHOT FUNCTION =====
    async function takeScreenshot() {
        if (!screenshotArea) {
            console.error("Screenshot area not found!");
            return;
        }

        try {
            const canvas = await html2canvas(screenshotArea, {
                scrollY: -window.scrollY,
                onclone: (clonedDocument) => {
                    const screenshotContainer =
                        clonedDocument.getElementById("screenshot-area");
                    if (screenshotContainer) {
                        screenshotContainer.style.paddingBottom = "2em";
                        screenshotContainer.style.backgroundColor = "#121212";
                    }

                    clonedDocument.body.style.backgroundColor = "#121212";
                    clonedDocument.documentElement.style.backgroundColor =
                        "#121212";
                    clonedDocument.body.style.color = "white";
                    clonedDocument.body.style.webkitTextFillColor = "white";

                    const h1Element = clonedDocument.querySelector("h1");
                    if (h1Element) {
                        h1Element.style.color = "#ff3c3c";
                        h1Element.style.webkitTextFillColor = "#ff3c3c";
                        h1Element.style.backgroundImage = "none";
                    }

                    const svgElement = clonedDocument.querySelector("svg");
                    if (svgElement) {
                        const rect = svgElement.querySelector("rect");
                        if (rect) {
                            rect.style.stroke = "white";
                            rect.style.fill = "none";
                        }

                        const circles = svgElement.querySelectorAll("circle");
                        circles.forEach((circle) => {
                            circle.style.opacity = "1";
                        });

                        const svgText = svgElement.querySelectorAll("text");
                        svgText.forEach((text) => {
                            text.style.fill = "white";
                        });

                        const svgLines = svgElement.querySelectorAll("line");
                        svgLines.forEach((line) => {
                            line.style.stroke = "#ff3c3c";
                            line.style.strokeWidth = "0.3";
                        });

                        const svgPaths = svgElement.querySelectorAll("path");
                        svgPaths.forEach((path) => {
                            path.style.stroke = "#ff3c3c";
                            path.style.strokeWidth = "0.3";
                        });
                    }

                    const allTextElements = clonedDocument.querySelectorAll(
                        "p, span, div, button, a, li, ul, strong, .current-container",
                    );
                    allTextElements.forEach((el) => {
                        if (el !== h1Element) {
                            el.style.color = "var(--text-color";
                            el.style.webkitTextFillColor =
                                "var(--background-color)";
                            el.style.backgroundImage = "none";
                        }
                    });

                    const lines =
                        clonedDocument.querySelectorAll(".menu-icon .line");
                    lines.forEach((line) => {
                        line.style.backgroundColor = "white";
                    });

                    const hr = clonedDocument.querySelector("hr");
                    if (hr) {
                        hr.style.borderColor = "white";
                    }

                    const urlElement = clonedDocument.createElement("div");
                    urlElement.textContent = url;
                    urlElement.style.position = "absolute";
                    urlElement.style.bottom = "5px";
                    urlElement.style.left = "0";
                    urlElement.style.right = "0";
                    urlElement.style.textAlign = "center";
                    urlElement.style.color = "white";
                    urlElement.style.fontWeight = "bold";
                    urlElement.style.fontSize = "18px";
                    urlElement.style.fontFamily = "Arial, sans-serif";
                    urlElement.style.zIndex = "9999";
                    urlElement.style.textShadow = "0 0 3px rgba(0,0,0,0.8)";
                    urlElement.style.backgroundColor = "rgba(150, 0, 0, 0.8)";
                    urlElement.style.padding = "8px 16px";
                    urlElement.style.borderRadius = "12px 12px 0 0";
                    urlElement.style.backdropFilter = "blur(2px)";
                    urlElement.style.margin = "-10px auto";
                    urlElement.style.width = "fit-content";
                    urlElement.style.maxWidth = "90%";
                    urlElement.style.border =
                        "1px solid rgba(255, 255, 255, 0.2)";

                    if (screenshotContainer) {
                        screenshotContainer.style.position = "relative";
                        screenshotContainer.appendChild(urlElement);
                    }
                },
            });

            const isMobile = /Mobi|Android/i.test(navigator.userAgent);

            if (isMobile) {
                const blob = await new Promise((resolve) =>
                    canvas.toBlob(resolve),
                );
                const url = URL.createObjectURL(blob);
                const newTab = window.open(url, "_blank");
                if (newTab) {
                    newTab.onload = () => {
                        showCustomAlert("Long-press the image to save it!");
                    };
                } else {
                    showCustomAlert(
                        "Pop-up blocker may have prevented the screenshot from opening.",
                    );
                }
            } else {
                const blob = await new Promise((resolve) =>
                    canvas.toBlob(resolve),
                );
                try {
                    const item = new ClipboardItem({ "image/png": blob });
                    await navigator.clipboard.write([item]);
                    showCustomAlert("Screenshot copied to clipboard!");
                } catch (err) {
                    console.warn("Clipboard write failed:", err);
                    showCustomAlert(
                        "Failed to copy to clipboard. Downloading instead.",
                    );
                }

                const today = new Date();
                const formattedDate = today.toISOString().split("T")[0];
                const filename = `alphabox_${formattedDate}.png`;
                const url = URL.createObjectURL(blob);
                const a = document.createElement("a");
                a.href = url;
                a.download = filename;
                document.body.appendChild(a);
                a.click();
                document.body.removeChild(a);
                URL.revokeObjectURL(url);
            }
        } catch (err) {
            console.error("Screenshot process failed:", err);
            showCustomAlert("Failed to generate screenshot.");
        }
    }

    // ===== INITIALIZATION CODE =====
    // Initialize puzzle geometry
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

    // Initialize game state
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

    // Set up caret blinking
    setInterval(() => {
        caret = caret ? "" : "|";
    }, 500);

    // Set up event listeners
    document.addEventListener("keydown", handleEscapeKey);
    document.addEventListener("keydown", function (event) {
        if (event.key == "Enter") {
            enterWord();
        } else if (event.key == "Backspace") {
            deleteLetter();
        } else if (event.altKey && event.shiftKey && event.key === "W") {
            // Alt+Shift+W to manually force a win
            manuallyForceWin();
            event.preventDefault();
        } else if (event.altKey && event.shiftKey && event.key === "I") {
            // Alt+Shift+I to manually increment streak
            manuallyIncrementStreak();
            event.preventDefault();
        } else if (event.altKey && event.shiftKey && event.key === "V") {
            // Alt+Shift+V to verify streak storage
            verifyStreakStorage();
            event.preventDefault();
        } else if (event.altKey && event.shiftKey && event.key === "R") {
            // Alt+Shift+R to reset streak
            resetStreak();
            event.preventDefault();
        } else {
            let i = letters.indexOf(event.key.toUpperCase());
            if (i != -1) selectLetter(i);
        }
    });

    function manuallyIncrementStreak() {
        // Get current streak data
        const currentStreak = getStreakData() || {
            current: 0,
            lastPlayed: null,
            longest: 0,
        };
        const today = getDate();

        console.log("Current streak before:", currentStreak);

        // Create updated streak data
        const updatedStreak = {
            current: currentStreak.current + 1,
            lastPlayed: today,
            longest: Math.max(currentStreak.current + 1, currentStreak.longest),
        };

        // Save to localStorage
        localStorage.setItem("alphabox-streak", JSON.stringify(updatedStreak));

        // Update the streakData variable to trigger UI refresh
        streakData = updatedStreak;

        console.log("Streak after increment:", updatedStreak);

        // Show the streak modal AFTER the UI has updated
        setTimeout(() => {
            showCustomAlert(
                `Streak manually incremented to ${updatedStreak.current} days!`,
            );
            showStreakModal = true;
        }, 50); // Short delay to ensure UI updates first

        return updatedStreak;
    }


    // Add theme preference variable
    let userThemePreference = localStorage.getItem("theme-preference") || "system";
    
    // Function to toggle theme
    function toggleTheme() {
        if (userThemePreference === "system") {
            userThemePreference = "dark";
        } else if (userThemePreference === "dark") {
            userThemePreference = "light";
        } else {
            userThemePreference = "system";
        }
        
        localStorage.setItem("theme-preference", userThemePreference);
        applyThemePreference();
    }
    
    // Function to apply theme preference
    function applyThemePreference() {
        if (userThemePreference === "dark") {
            document.documentElement.classList.add("force-dark");
            document.documentElement.classList.remove("force-light");
        } else if (userThemePreference === "light") {
            document.documentElement.classList.add("force-light");
            document.documentElement.classList.remove("force-dark");
        } else {
            document.documentElement.classList.remove("force-dark", "force-light");
        }
    }


</script>

<main>
    <!-- This is for testing streak saving. It can be removed once we are 100% certain localStorage works well.
    <button
        on:click={manuallyIncrementStreak}
        style="position: fixed; bottom: 10px; left: 10px; z-index: 9999; opacity: 0.5; font-size: 10px; padding: 2px 5px;"
        title="Debug: Increment Streak"
    >
        🎯 +1
    </button> -->
    {#if showStreakModal}
        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <div
            class="modal"
            on:click={() => (showStreakModal = false)}
            transition:fade={{ duration: 250 }}
        >
            <div
                class="modal-content"
                on:click|stopPropagation
                transition:fly={{ y: -20, duration: 300 }}
            >
                <button
                    class="close-button"
                    on:click={() => (showStreakModal = false)}>&times;</button
                >
                <h3>Daily AlphaBox</h3>
                <div class="streak-container">
                    {#if streakData && streakData.current > 0}
                        <div class="streak-badge">
                            <span class="streak-count"
                                >{streakData.current}</span
                            >
                            <span class="streak-label">day streak! 🔥</span>
                        </div>
                        <p>
                            Your longest streak: <strong
                                >{streakData.longest} days</strong
                            >
                        </p>
                    {:else}
                        <div class="streak-badge">
                            <span class="streak-label"
                                >Time to start a streak!</span
                            >
                        </div>
                        <p>
                            Your longest streak: <strong
                                >{streakData ? streakData.longest : 0} days</strong
                            >
                        </p>
                    {/if}

                    <!-- Add today's puzzle info -->
                    <div class="puzzle-info">
                        <p>
                            Today's puzzle has {easySolutionCount} optimal solution{easySolutionCount !==
                            1
                                ? "s"
                                : ""}
                        </p>
                        <p>Try to solve it in {par} or fewer words!</p>
                    </div>
                </div>

                <!-- Simple View Board button that just closes the modal -->
                <button
                    class="streak-continue-button"
                    on:click={() => (showStreakModal = false)}
                >
                    View Board
                </button>
            </div>
        </div>
    {/if}
    {#if !showMenu}
        <button
            class="menu-button {showMenu ? 'sidebar-open' : ''}"
            on:click={() => (showMenu = !showMenu)}
            on:mouseenter={openSidebarOnHover}
            on:mouseleave={closeSidebarOnLeave}
            in:fade={{ duration: 200 }}
            out:fade={{ duration: 200 }}
        >
            <div class="menu-icon">
                <div class="line"></div>
                <div class="line"></div>
                <div class="line"></div>
            </div>
        </button>
    {/if}

    <!-- Sidebar + overlay -->
    {#if showMenu}
        <div class="sidebar-overlay">
            <!-- Background overlay -->
            <!-- svelte-ignore a11y-click-events-have-key-events -->
            <div
                class="overlay-background"
                on:click={() => (showMenu = false)}
                in:fade={{ duration: 200 }}
                out:fade={{ duration: 200 }}
            ></div>

            <!-- Sliding sidebar -->
            <aside
                class="sidebar"
                in:fly={{ x: -300, duration: 250, easing: cubicOut }}
                out:fly={{ x: -300, duration: 250, easing: cubicIn }}
                on:mouseenter={() => setSidebarHover(true)}
                on:mouseleave={() => setSidebarHover(false)}
            >
                <div class="sidebar-content">
                    <button
    class="sidebar-link"
    on:click={() => {
        toggleTheme();
        showMenu = false;
    }}
>
    {#if userThemePreference === 'dark'}
        <svg class="sidebar-icon" viewBox="0 0 24 24" fill="currentColor">
            <path d="M9.37 5.51A7.35 7.35 0 0 0 9.1 7.5c0 4.08 3.32 7.4 7.4 7.4.68 0 1.35-.09 1.99-.27A7.014 7.014 0 0 1 12 19c-3.86 0-7-3.14-7-7 0-2.93 1.81-5.45 4.37-6.49z"/>
        </svg>
        <span class="glow-text">Dark Theme</span>
    {:else if userThemePreference === 'light'}
        <svg class="sidebar-icon" viewBox="0 0 24 24" fill="currentColor">
            <path d="M12 7c-2.76 0-5 2.24-5 5s2.24 5 5 5 5-2.24 5-5-2.24-5-5-5zM2 13h2c.55 0 1-.45 1-1s-.45-1-1-1H2c-.55 0-1 .45-1 1s.45 1 1 1zm18 0h2c.55 0 1-.45 1-1s-.45-1-1-1h-2c-.55 0-1 .45-1 1s.45 1 1 1zM11 2v2c0 .55.45 1 1 1s1-.45 1-1V2c0-.55-.45-1-1-1s-1 .45-1 1zm0 18v2c0 .55.45 1 1 1s1-.45 1-1v-2c0-.55-.45-1-1-1s-1 .45-1 1zM6.34 5.16l-1.42 1.42c-.39.39-.39 1.02 0 1.41.39.39 1.02.39 1.41 0l1.42-1.42c.39-.39.39-1.02 0-1.41a.9959.9959 0 0 0-1.41 0zm13.08 12.42l1.42 1.42c.39.39 1.02.39 1.41 0 .39-.39.39-1.02 0-1.41l-1.42-1.42c-.39-.39-1.02-.39-1.41 0a.9959.9959 0 0 0 0 1.41zM5.16 17.66l1.42 1.42c.39.39 1.02.39 1.41 0 .39-.39.39-1.02 0-1.41L6.57 16.25c-.39-.39-1.02-.39-1.41 0a.9959.9959 0 0 0 0 1.41zm12.42-13.08l1.42-1.42c.39-.39.39-1.02 0-1.41-.39-.39-1.02-.39-1.41 0l-1.42 1.42c-.39.39-.39 1.02 0 1.41.39.39 1.02.39 1.41 0z"/>
        </svg>
        <span class="glow-text">Light Theme</span>
    {:else}
        <svg class="sidebar-icon" viewBox="0 0 24 24" fill="currentColor">
            <path d="M20 8.69V4h-4.69L12 .69 8.69 4H4v4.69L.69 12 4 15.31V20h4.69L12 23.31 15.31 20H20v-4.69L23.31 12 20 8.69zM12 18c-3.31 0-6-2.69-6-6s2.69-6 6-6 6 2.69 6 6-2.69 6-6 6zm0-10c-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4-1.79-4-4-4z"/>
        </svg>
        <span class="glow-text">System Theme</span>
    {/if}
</button>
                    <button
                        class="sidebar-link"
                        on:click={() => {
                            help = true;
                            showMenu = false;
                        }}
                    >
                        <svg
                            class="sidebar-icon"
                            viewBox="0 0 24 24"
                            fill="currentColor"
                        >
                            <path
                                d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 17h-2v-2h2v2zm2.07-7.75l-.9.92C13.45 12.9 13 13.5 13 15h-2v-.5c0-1.1.45-2.1 1.17-2.83l1.24-1.26c.37-.36.59-.86.59-1.41 0-1.1-.9-2-2-2s-2 .9-2 2H8c0-2.21 1.79-4 4-4s4 1.79 4 4c0 .88-.36 1.68-.93 2.25z"
                            />
                        </svg>
                        <span class="glow-text">How to Play</span>
                    </button>

                    {#if !yesterdayLoaded}
                        <button
                            class="sidebar-link"
                            on:click={() => {
                                loadYesterdayPuzzle();
                                showMenu = false;
                            }}
                        >
                            <svg
                                class="sidebar-icon"
                                viewBox="0 0 24 24"
                                fill="currentColor"
                            >
                                <path
                                    d="M20 11H7.83l5.59-5.59L12 4l-8 8 8 8 1.41-1.41L7.83 13H20v-2z"
                                />
                            </svg>
                            <span class="glow-text">Yesterday's Game</span>
                        </button>
                    {:else}
                        <button
                            class="sidebar-link"
                            on:click={() => {
                                loadTodayPuzzle();
                                showMenu = false;
                            }}
                        >
                            <svg
                                class="sidebar-icon"
                                viewBox="0 0 24 24"
                                fill="currentColor"
                            >
                                <path
                                    d="M19 3h-1V1h-2v2H8V1H6v2H5c-1.11 0-1.99.9-1.99 2L3 19c0 1.1.89 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm0 16H5V8h14v11zM7 10h5v5H7z"
                                />
                            </svg>
                            <span class="glow-text">Today's Game</span>
                        </button>
                    {/if}

                    <button
                        class="sidebar-link"
                        on:click={() => {
                            loadRandomPuzzle();
                            showMenu = false;
                        }}
                    >
                        <svg
                            class="sidebar-icon"
                            viewBox="0 0 24 24"
                            fill="currentColor"
                        >
                            <path
                                d="M20 6h-2.18c.11-.31.18-.65.18-1 0-1.66-1.34-3-3-3-1.05 0-1.96.54-2.5 1.35l-.5.67-.5-.68C10.96 2.54 10.05 2 9 2 7.34 2 6 3.34 6 5c0 .35.07.69.18 1H4c-1.11 0-1.99.89-1.99 2L2 19c0 1.11.89 2 2 2h16c1.11 0 2-.89 2-2V8c0-1.11-.89-2-2-2zm-5-2c.55 0 1 .45 1 1s-.45 1-1 1-1-.45-1-1 .45-1 1-1zM9 4c.55 0 1 .45 1 1s-.45 1-1 1-1-.45-1-1 .45-1 1-1zm11 15H4v-2h16v2zm0-5H4V8h5.08L7 10.83 8.62 12 12 7.4l3.38 4.6L17 10.83 14.92 8H20v6z"
                            />
                        </svg>
                        <span class="glow-text">Random Game</span>
                    </button>

                    {#if !yesterdayLoaded}
                        <button
                            class="sidebar-link"
                            on:click={() => {
                                let format = (s) => `${s[0]} - ${s[1]}`;
                                solutions = solve(letters).map(format);
                                allSolutions = solveAll(letters).map(format);
                                solutionsModal = true;
                                showMenu = false;
                            }}
                        >
                            <svg
                                class="sidebar-icon"
                                viewBox="0 0 24 24"
                                fill="currentColor"
                            >
                                <path
                                    d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 17h-2v-2h2v2zm2.07-7.75l-.9.92C13.45 12.9 13 13.5 13 15h-2v-.5c0-1.1.45-2.1 1.17-2.83l1.24-1.26c.37-.36.59-.86.59-1.41 0-1.1-.9-2-2-2s-2 .9-2 2H8c0-2.21 1.79-4 4-4s4 1.79 4 4c0 .88-.36 1.68-.93 2.25z"
                                />
                            </svg>
                            <span class="glow-text">View Solutions</span>
                        </button>
                    {:else}
                        <button
                            class="sidebar-link"
                            on:click={() => {
                                let yesterdayPuzzle = generate(yesterday())
                                    .join("")
                                    .toUpperCase();
                                let format = (s) => `${s[0]} - ${s[1]}`;
                                solutions = solve(yesterdayPuzzle).map(format);
                                allSolutions =
                                    solveAll(yesterdayPuzzle).map(format);
                                solutionsModal = true;
                                showMenu = false;
                            }}
                        >
                            <svg
                                class="sidebar-icon"
                                viewBox="0 0 24 24"
                                fill="currentColor"
                            >
                                <path
                                    d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 17h-2v-2h2v2zm2.07-7.75l-.9.92C13.45 12.9 13 13.5 13 15h-2v-.5c0-1.1.45-2.1 1.17-2.83l1.24-1.26c.37-.36.59-.86.59-1.41 0-1.1-.9-2-2-2s-2 .9-2 2H8c0-2.21 1.79-4 4-4s4 1.79 4 4c0 .88-.36 1.68-.93 2.25z"
                                />
                            </svg>
                            <span class="glow-text">View Solutions</span>
                        </button>
                    {/if}
                </div>
            </aside>
        </div>
    {/if}

    {#if help}
        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <div
            class="modal"
            on:click={() => {
                // Only allow closing by clicking outside for non-first-time users
                if (!showHelpOnLoad) {
                    help = false;
                }
            }}
        >
            <div class="modal-content" on:click|stopPropagation>
                {#if !showHelpOnLoad}
                    <button class="close-button" on:click={() => (help = false)}
                        >&times;</button
                    >
                {/if}
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
                        <a href={githubUrl} target="_blank"
                            >Source code and trivia</a
                        >
                    </p>
                </ul>
                {#if showHelpOnLoad}
                    <button
                        class="streak-continue-button"
                        on:click={() => {
                            help = false;
                            showHelpOnLoad = false;
                            // Initialize streak data for first-time user
                            streakData = updateStreak(false);
                        }}
                    >
                        Let's Play!
                    </button>
                {/if}

                <h3 style="padding-top: 1em;">How "Par" is Calculated</h3>
                <p>
                    While there is always a two-word solution to the puzzle,
                    some games may be more difficult to figure out than others.
                    To that end, and to make the game more casual, a "par"
                    system has been put in place to give a realistic expectation
                    of how many guesses you should complete the puzzle in when
                    playing casually.
                </p>
                <ul>
                    <li>
                        Each puzzle has at least one way to complete it with two
                        words that only share one letter (the connecting
                        letter.)
                    </li>
                    <li>
                        Puzzles can still be completed with repeating letters -
                        they just aren't the optimal solutions.
                    </li>
                    <li>
                        If there is only one optimal solution and it is the only
                        possible two-word solution, par is set at 6.
                    </li>
                    <li>
                        If there is only one optimal solution and there are
                        fewer than 4 possible two-word solutions, par is set to
                        5.
                    </li>
                    <li>
                        If there is only one optimal solution and there are 4 or
                        more possible two-word solutions OR if there are 2
                        possible optimal solutions, par is set to 4.
                    </li>
                    <li>
                        If there are 3 or more optimal solutions, par is set to
                        3.
                    </li>
                </ul>
            </div>
        </div>
    {/if}

    {#if solutionsModal}
        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <div
            class="modal"
            bind:this={modal}
            on:click={(evt) => {
                solutionsModal = evt.target != modal;
            }}
        >
            <div class="modal-content">
                <button
                    class="close-button"
                    on:click={() => (solutionsModal = false)}>&times;</button
                >
                <h3>Solutions</h3>
                <div class="solutions-container">
                    <p>
                        This list only contains solutions which do not repeat
                        any letters except the connecting letter.
                    </p>
                    {#each solutions as sol}
                        <p style="font-weight: bold">{sol}</p>
                    {/each}
                    {#each allSolutions as sol}
                        {#if !solutions.includes(sol)}
                            <p>{sol}</p>
                        {/if}
                    {/each}
                </div>
            </div>
        </div>
    {/if}

    <div id="screenshot-area" style="width: auto;" bind:this={screenshotArea}>
        <h1>alphabox</h1>

        <p class="par">
            Try to get it in <strong>{par}</strong> or fewer words!
        </p>
        <div class="current">
            {#if done}
                <div class="solved-words">
                    {previousWords.join(" - ")}
                </div>
                <hr
                    style="min-width: 10em; max-width: 13em;
            border:1px solid var(--svg-stroke-color); margin-top: 0"
                />
            {:else}
                <span class="current-container">
                    <span
                        class:shake={shakeAnimation}
                        bind:this={currentEl}
                        style="display: inline-block; 
                 transform-origin: left center;
                 white-space: nowrap;"
                    >
                        {currentText}
                    </span>
                    <span class="caret">{caret}</span>
                </span>
                <hr
                    style="min-width: 10em; max-width: 13em;
            border:1px solid var(--svg-stroke-color); margin-top: 0"
                />
            {/if}
        </div>

        <svg
            xmlns="http://www.w3.org/2000/svg"
            viewBox="0 0 42 44"
            style="margin-bottom: -1em; margin-top: -.25em;"
        >
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
                            x2={circles[revindex(letters.indexOf(word[i + 1]))]
                                .x}
                            y2={circles[revindex(letters.indexOf(word[i + 1]))]
                                .y}
                            stroke={pos == previousWords.length
                                ? "#ff3e00"
                                : "#ff3e0080"}
                            stroke-width={stroke}
                            stroke-dasharray={pos == previousWords.length
                                ? 2
                                : 0}
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
                    fill={letterColor(index(i), lastLetter, currentWord)}
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
                <!-- svelte-ignore a11y-click-events-have-key-events -->
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
    </div>
    <!-- svelte-ignore a11y-click-events-have-key-events -->
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
                <!-- svelte-ignore a11y-missing-attribute -->
                <a class="title">View on GitHub</a>
            </div>
        </div>
    </div>
    {#if done}
        <div class="screenshot-buttons">
            <button on:click={takeScreenshot}>Share Results</button>
        </div>
        <div class="confetti-container">
            {#each Array(150) as _, i}
                <div
                    class="confetti-piece"
                    style="
    --spread: {Math.random() * 100 - 50}vw;
    --wobble: {Math.random() * 30 - 15}px; /* -15px to +15px wobble */
    background: hsl({Math.random() * 360}, 100%, 50%);
    animation-duration: 2.8s;
    animation-delay: {Math.random() * 0.4}s;
    border-radius: {i % 3 === 0 ? '50%' : '0'};
  "
                ></div>
            {/each}
        </div>
    {:else}
        <div class="buttons">
            <button on:click={deleteLetter}>Delete</button>
            <button on:click={enterWord}>Submit</button>
        </div>
    {/if}
</main>

<style>
</style>
