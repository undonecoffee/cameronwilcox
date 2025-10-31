<script>
    import { onMount, onDestroy } from "svelte";

    // === State ===
    let words = [];
    let normalWords = [];
    let hardWords = [];
    let unfilteredWords = [];

    // Only new words added this session
    let newNormalWords = [];
    let newHardWords = [];

    let currentWord = "";
    let input = "";
    let previousWord = null;
    let previousStatus = null;

    // === Load JSON files ===
    async function loadWordLists() {
        const [wordsRes, normalRes, hardRes] = await Promise.all([
            fetch("/words.json"),
            fetch("/normalWords.json"),
            fetch("/hardWords.json"),
        ]);

        words = await wordsRes.json();
        normalWords = await normalRes.json();
        hardWords = await hardRes.json();

        filterWords();
        newWord();
    }

    // === Filter out completed words ===
    function filterWords() {
        unfilteredWords = words.filter(
            (w) => !normalWords.includes(w) && !hardWords.includes(w),
        );
    }

    // === Pick a new random word ===
    function newWord() {
        if (unfilteredWords.length === 0) {
            currentWord = "";
            return;
        }
        const idx = Math.floor(Math.random() * unfilteredWords.length);
        currentWord = unfilteredWords[idx];
        input = "";
        speak(currentWord);
    }

    // === Speak the word ===
    function speak(word) {
        const utter = new SpeechSynthesisUtterance(word);
        utter.rate = 1.1;
        speechSynthesis.speak(utter);
    }

    // === Confirm typed word ===
    function confirmWord() {
        if (!currentWord) return;

        if (input.trim().toLowerCase() === currentWord.toLowerCase()) {
            if (
                !normalWords.includes(currentWord) &&
                !hardWords.includes(currentWord)
            ) {
                normalWords = [...normalWords, currentWord];
                newNormalWords = [...newNormalWords, currentWord];
                saveList("normalWords.json", normalWords);
            }

            previousWord = currentWord;
            previousStatus = "correct";
            unfilteredWords = unfilteredWords.filter((w) => w !== currentWord);
            newWord();
        } else {
            previousWord = currentWord;
            previousStatus = "wrong";
            speak(currentWord);
            input = "";
        }
    }

    // === Restart current word ===
    function restartWord() {
        if (currentWord) speak(currentWord);
    }

    // === Mark word as hard ===
    function markHard(word) {
        if (!word) return;
        normalWords = normalWords.filter((w) => w !== word);
        newNormalWords = newNormalWords.filter((w) => w !== word);

        if (!hardWords.includes(word)) {
            hardWords = [...hardWords, word];
            newHardWords = [...newHardWords, word];
        }

        saveList("normalWords.json", normalWords);
        saveList("hardWords.json", hardWords);
        filterWords();
    }

    function saveList(name, data) {
        localStorage.setItem(name, JSON.stringify(data));
    }

    async function copyToClipboard(list) {
        const text = list.map((w) => `"${w}",`).join("\n");
        await navigator.clipboard.writeText(text);
    }

    // === Keyboard events ===
    function handleKeydown(e) {
        if (e.code === "Space") {
            e.preventDefault();

            if (previousStatus === "correct" && input.trim() === "") {
                markHard(previousWord);
                return;
            }

            confirmWord();
        }

        if (e.code === "Enter") {
            e.preventDefault();
            restartWord();
        }
    }

    onMount(async () => {
        await loadWordLists();
        window.addEventListener("keydown", handleKeydown);
    });
</script>

<main>
    <div class="container">
        <h1>trainer</h1>
        <p class="counter">{unfilteredWords.length} left</p>

        {#if previousWord}
            <div class="previous {previousStatus}">
                {previousWord}
            </div>
        {/if}

        <!-- input stays centered / fixed -->
        <div class="input-area">
            <input bind:value={input} placeholder="type here..." autofocus />
        </div>

        <!-- scrollable infinite lists -->
        <div class="lists">
            <div class="list">
                <div class="list-header">
                    <span>normal ({newNormalWords.length})</span>
                    {#if newNormalWords.length > 0}
                        <button on:click={() => copyToClipboard(newNormalWords)}
                            >copy</button
                        >
                    {/if}
                </div>
                {#if newNormalWords.length > 0}
                    <pre>{newNormalWords.map((w) => `"${w}",`).join("\n")}</pre>
                {/if}
            </div>

            <div class="list">
                <div class="list-header">
                    <span>hard ({newHardWords.length})</span>
                    {#if newHardWords.length > 0}
                        <button on:click={() => copyToClipboard(newHardWords)}
                            >copy</button
                        >
                    {/if}
                </div>
                {#if newHardWords.length > 0}
                    <pre>{newHardWords.map((w) => `"${w}",`).join("\n")}</pre>
                {/if}
            </div>
        </div>
    </div>
</main>

<style>
    :root {
        --bg: #222;
        --text: #ccc;
        --accent: #999;
        --green: #81c784;
        --red: #e57373;
    }

    main {
        position: relative;
        display: flex;
        justify-content: center;
        background: var(--bg);
        color: var(--text);
        font-family: "Fira Code", monospace;
        min-height: 100vh;
        overflow: hidden;
    }

    .container {
        position: relative;
        width: 400px;
        text-align: center;
        display: flex;
        flex-direction: column;
        align-items: center;
    }

    h1 {
        margin: 0;
        font-weight: 400;
        color: var(--accent);
        opacity: 0.8;
        letter-spacing: 1px;
        padding-top: 2rem;
    }

    .counter {
        font-size: 0.85rem;
        margin-bottom: 0.5rem;
        color: #777;
    }

    .previous {
        font-size: 1.2rem;
        margin-bottom: 1rem;
    }

    .previous.correct {
        color: var(--green);
    }

    .previous.wrong {
        color: var(--red);
    }

    /* Centered input box */
    .input-area {
        position: fixed;
        top: 16.5%;
        left: 50%;
        transform: translate(-50%, -50%);
        width: 300px;
        z-index: 5;
    }

    input {
        width: 100%;
        font-size: 1.4rem;
        text-align: center;
        background: transparent;
        border: none;
        border-bottom: 1px solid #444;
        color: var(--text);
        outline: none;
        transition: border 0.2s;
    }

    input:focus {
        border-bottom: 1px solid var(--accent);
    }

    /* Scrollable lists section */
    .lists {
        position: absolute;
        top: 20%;
        left: 55%;
        transform: translateX(-50%);
        width: 90%;
        max-width: 500px;
        display: flex;
        gap: 1rem;
        justify-content: space-between;
        overflow-y: auto;
        padding-bottom: 2rem;
    }

    .list {
        flex: 1;
        text-align: left;
    }

    .list-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        font-size: 0.8rem;
        color: var(--accent);
        margin-bottom: 0.4rem;
    }

    button {
        background: none;
        color: var(--accent);
        border: none;
        font-size: 0.7rem;
        cursor: pointer;
        opacity: 0.6;
    }

    button:hover {
        opacity: 1;
    }

    pre {
        margin: 0;
        font-size: 0.75rem;
        color: #888;
        white-space: pre-wrap;
        word-break: break-all;
    }
</style>
