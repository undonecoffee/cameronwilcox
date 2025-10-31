<script context="module">
    export const ssr = false; // keep SSR off for speech + window usage
</script>

<script>
    import { onMount, onDestroy } from "svelte";

    let words = [];
    let existingHardWords = [];
    let existingNormalWords = [];
    let availableWords = [];

    let currentWord = "";
    let input = "";
    let previousWord = null;
    let previousStatus = null; // 'correct' | 'wrong'
    let hardWords = [];
    let normalWords = [];

    let showCopied = false;
    let copyType = ""; // show which list we copied

    // Load all lists from /static/
    async function loadData() {
        try {
            const [wordsRes, hardRes, normalRes] = await Promise.all([
                fetch("/words.json"),
                fetch("/hardWords.json"),
                fetch("/normalWords.json"),
            ]);

            words = await wordsRes.json();
            existingHardWords = await hardRes.json();
            existingNormalWords = await normalRes.json();

            // filter out words already categorized
            availableWords = words.filter(
                (w) =>
                    !existingHardWords.includes(w) &&
                    !existingNormalWords.includes(w),
            );

            newWord();
        } catch (err) {
            console.error("Failed to load word lists:", err);
        }
    }

    function newWord() {
        if (availableWords.length === 0) {
            currentWord = "";
            alert("No more words left!");
            return;
        }

        const idx = Math.floor(Math.random() * availableWords.length);
        currentWord = availableWords[idx];
        speak(currentWord);
        input = "";
    }

    function speak(word) {
        const utter = new SpeechSynthesisUtterance(word);
        utter.rate = 0.8;
        speechSynthesis.speak(utter);
    }

    function confirmWord() {
        if (!currentWord) return;

        if (input.trim().toLowerCase() === currentWord.toLowerCase()) {
            previousWord = currentWord;
            previousStatus = "correct";
            availableWords = availableWords.filter((w) => w !== currentWord);
            newWord();
        } else {
            previousWord = currentWord;
            previousStatus = "wrong";
            speak(currentWord);
            input = "";
        }
    }

    function restartWord() {
        if (currentWord) speak(currentWord);
    }

    // Add to hard or normal words
    function addToList(list, word, existingList) {
        if (!word || list.includes(word) || existingList.includes(word)) return;
        list = [...list, word]; // clone to trigger reactivity
        return list;
    }

    function handleKeydown(e) {
        if (e.code === "Space" && e.ctrlKey) {
            // Ctrl + Space → add to hard
            e.preventDefault();
            if (previousStatus === "correct" && input.trim() === "") {
                hardWords = addToList(
                    hardWords,
                    previousWord,
                    existingHardWords,
                );
                console.log("Added to hard words:", previousWord);
                return;
            }
        }

        if (e.code === "Space" && !e.ctrlKey) {
            e.preventDefault();

            // Normal space → add to normal words
            if (previousStatus === "correct" && input.trim() === "") {
                normalWords = addToList(
                    normalWords,
                    previousWord,
                    existingNormalWords,
                );
                console.log("Added to normal words:", previousWord);
                return;
            }

            confirmWord();
        }

        if (e.code === "Enter") {
            e.preventDefault();
            restartWord();
        }
    }

    // Copy formatted lists
    function copyList(type) {
        let list = type === "hard" ? hardWords : normalWords;
        if (!list || list.length === 0) return;
        const formatted = list.map((w) => `"${w}",`).join("\n");
        navigator.clipboard.writeText(formatted);
        copyType = type;
        showCopied = true;
        setTimeout(() => (showCopied = false), 1500);
    }

    onMount(() => {
        loadData();
        window.addEventListener("keydown", handleKeydown);
    });

    onDestroy(() => {
        window.removeEventListener("keydown", handleKeydown);
    });
</script>

<main>
    {#if previousWord}
        <div class="previous {previousStatus}">
            {previousWord}
        </div>
    {/if}

    <input bind:value={input} placeholder="Type the word..." autofocus />

    <div class="lists">
        <div class="listbox">
            <h2>Normal Words</h2>
            {#if normalWords.length > 0}
                <ul>
                    {#each normalWords as word}
                        <li>{word}</li>
                    {/each}
                </ul>
            {:else}
                <p class="empty">None yet</p>
            {/if}
            <button
                on:click={() => copyList("normal")}
                disabled={normalWords.length === 0}
            >
                Copy Normal
            </button>
        </div>

        <div class="listbox">
            <h2>Hard Words</h2>
            {#if hardWords.length > 0}
                <ul>
                    {#each hardWords as word}
                        <li>{word}</li>
                    {/each}
                </ul>
            {:else}
                <p class="empty">None yet</p>
            {/if}
            <button
                on:click={() => copyList("hard")}
                disabled={hardWords.length === 0}
            >
                Copy Hard
            </button>
        </div>
    </div>

    {#if showCopied}
        <div class="copied">Copied {copyType} words!</div>
    {/if}

    <div class="instructions">
        Space: add to normal | Ctrl + Space: add to hard | Enter: repeat
    </div>
</main>

<style>
    main {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        height: 100vh;
        background: #1b1b1b;
        color: #eee;
        font-family: "Fira Code", monospace;
    }

    .previous {
        font-size: 1.5rem;
        margin-bottom: 1rem;
    }

    .previous.correct {
        color: #00c853;
    }

    .previous.wrong {
        color: #ff5252;
    }

    input {
        font-size: 2rem;
        text-align: center;
        background: transparent;
        border: none;
        border-bottom: 2px solid #555;
        color: white;
        outline: none;
        width: 300px;
    }

    input:focus {
        border-bottom: 2px solid #9acd32;
    }

    .instructions {
        position: absolute;
        bottom: 1rem;
        font-size: 0.9rem;
        color: #999;
    }

    .lists {
        position: absolute;
        top: 1rem;
        right: 1rem;
        display: flex;
        flex-direction: column;
        gap: 0.75rem;
    }

    .listbox {
        background: #222;
        border: 1px solid #333;
        padding: 0.75rem 1rem;
        border-radius: 6px;
        max-height: 220px;
        overflow-y: auto;
        font-size: 0.9rem;
        width: 220px;
    }

    .listbox h2 {
        margin: 0 0 0.5rem 0;
        font-size: 1rem;
        color: #9acd32;
    }

    .listbox ul {
        list-style: none;
        margin: 0;
        padding: 0;
    }

    .listbox li {
        margin: 0.2rem 0;
    }

    button {
        background: #333;
        color: #9acd32;
        border: 1px solid #555;
        padding: 0.4rem 0.8rem;
        border-radius: 4px;
        font-family: inherit;
        font-size: 0.8rem;
        cursor: pointer;
        transition: all 0.2s ease;
        width: 100%;
        margin-top: 0.5rem;
    }

    button:hover {
        background: #444;
    }

    button:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }

    .copied {
        position: absolute;
        bottom: 3rem;
        right: 1rem;
        background: #2e7d32;
        color: white;
        padding: 0.4rem 0.8rem;
        border-radius: 4px;
        font-size: 0.8rem;
        opacity: 0.9;
    }
</style>
