<script>
    import Tile from "../lib/tiles/Tile.svelte";

    const PRESETS = [
        { name: "Easy", size: 5, count: 5, rounds: 5 },
        { name: "D9", size: 5, count: 6, rounds: 6 },
        { name: "Hard", size: 6, count: 8, rounds: 7 },
        { name: "Prospero", size: 6, count: 10, rounds: 7 },
        { name: "Impossible", size: 8, count: 14, rounds: 8 },
    ];

    let selectedPreset = $state("");

    function applyPreset(e) {
        const idx = e.target.value;
        if (idx === "") return;
        const p = PRESETS[idx];
        gridSize = p.size;
        activeCount = p.count;
        roundCount = p.rounds;
    }

    function onManualChange() {
        selectedPreset = "";
    }

    let tileRefs = $state([]);
    let gridSize = $state(5);
    let activeCount = $state(5);
    let roundCount = $state(5);
    let showDuration = $state(1500);
    let clickDuration = $state(2500);
    let tileColor = $state("#3f8fe9");

    const totalTiles = $derived(gridSize * gridSize);

    let activeTiles = $state(new Set());
    let clickable = $state(false);
    let running = $state(false);
    let round = $state(0);
    let phase = $state("idle");

    let failResolve = null;

    let progress = $state(0);
    let progressColor = $state("#7c3aed");
    let progressRaf = null;

    function startProgress(duration, color) {
        progressColor = color;
        progress = 1;
        const start = performance.now();
        cancelAnimationFrame(progressRaf);
        function tick(now) {
            const elapsed = now - start;
            progress = Math.max(0, 1 - elapsed / duration);
            if (elapsed < duration) progressRaf = requestAnimationFrame(tick);
        }
        progressRaf = requestAnimationFrame(tick);
    }

    function stopProgress() {
        cancelAnimationFrame(progressRaf);
        progress = 0;
    }

    async function start() {
        if (running) return;
        running = true;
        phase = "idle";
        round = 1;

        for (let r = 0; r < roundCount; r++) {
            activeTiles = new Set();
            clickable = false;
            tileRefs.forEach((t) => t?.reset());

            const picked = new Set();
            while (picked.size < Math.min(activeCount, totalTiles)) {
                picked.add(Math.floor(Math.random() * totalTiles));
            }
            activeTiles = picked;

            phase = "showing";
            startProgress(showDuration, "#7c3aed");
            await Promise.all(
                [...picked].map((i) => tileRefs[i]?.show(showDuration)),
            );
            stopProgress();

            phase = "clicking";
            clickable = true;
            startProgress(clickDuration, "#a855f7");
            const failed = await raceFailure(clickDuration);
            stopProgress();
            clickable = false;

            round++;

            if (failed) {
                phase = "failed";
                for (const i of activeTiles) {
                    tileRefs[i]?.reveal();
                }
                running = false;
                return;
            }

            if (r < roundCount - 1) {
                phase = "idle";
                await wait(600);
            }
        }

        phase = "done";
        running = false;
    }

    let correctClicks = 0;

    function raceFailure(ms) {
        correctClicks = 0;
        return new Promise((resolve) => {
            failResolve = resolve;
            setTimeout(() => {
                resolve(correctClicks < activeTiles.size);
            }, ms);
        });
    }

    function handleTileClick(i, wasActive) {
        if (!wasActive && failResolve) {
            failResolve(true);
            failResolve = null;
        } else if (wasActive) {
            correctClicks++;
            if (correctClicks === activeTiles.size && failResolve) {
                failResolve(false);
                failResolve = null;
            }
        }
    }

    function wait(ms) {
        return new Promise((r) => setTimeout(r, ms));
    }

    const phaseLabel = $derived(
        phase === "showing"
            ? `Round ${round} of ${roundCount} — watch!`
            : phase === "clicking"
              ? `Round ${round} of ${roundCount} — click the pattern!`
              : phase === "failed"
                ? "❌"
                : phase === "done"
                  ? "✅"
                  : "",
    );

    const roundStatuses = $derived(
        Array.from({ length: roundCount }, (_, i) => {
            if (phase === "failed" && i === round - 2) return "failed";
            if (i < round - 1) return "done";
            if (i === round - 1 && phase !== "idle" && phase !== "failed")
                return "current";
            return "upcoming";
        }),
    );
</script>

<div class="controls">
    <label>
        Preset
        <select
            bind:value={selectedPreset}
            onchange={applyPreset}
            disabled={running}
        >
            <option value="">— custom —</option>
            {#each PRESETS as preset, i}
                <option value={i}>{preset.name}</option>
            {/each}
        </select>
    </label>

    <div class="divider"></div>

    <label>
        Grid size
        <input
            type="number"
            bind:value={gridSize}
            oninput={onManualChange}
            min="2"
            max="10"
            disabled={running}
        />
    </label>
    <label>
        Active tiles
        <input
            type="number"
            bind:value={activeCount}
            oninput={onManualChange}
            min="1"
            max={totalTiles}
            disabled={running}
        />
    </label>
    <label>
        Rounds
        <input
            type="number"
            bind:value={roundCount}
            oninput={onManualChange}
            min="1"
            max="20"
            disabled={running}
        />
    </label>
    <label>
        Show (ms)
        <input
            type="number"
            bind:value={showDuration}
            min="300"
            max="5000"
            step="100"
            disabled={running}
        />
    </label>
    <label>
        Click time (ms)
        <input
            type="number"
            bind:value={clickDuration}
            min="500"
            max="10000"
            step="500"
            disabled={running}
        />
    </label>
    <label>
        Tile color
        <div class="color-input">
            <input type="color" bind:value={tileColor} disabled={running} />
            <input
                type="text"
                bind:value={tileColor}
                maxlength="7"
                disabled={running}
                class="hex"
            />
        </div>
    </label>
    <button onclick={start} disabled={running}>
        {running ? "Running…" : phase === "idle" ? "Start" : "Restart"}
    </button>
</div>

<div class="progress-track">
    <div
        class="progress-bar"
        style="transform: scaleX({progress}); background: {progressColor};"
    ></div>
</div>

<div class="grid" style="--cols: {gridSize};">
    {#each { length: totalTiles } as _, i}
        <Tile
            bind:this={tileRefs[i]}
            active={activeTiles.has(i)}
            color={tileColor}
            {clickable}
            onClick={(wasActive) => handleTileClick(i, wasActive)}
        />
    {/each}
</div>

<div class="round-tracker" style="--tile-color: {tileColor};">
    {#each roundStatuses as status}
        <div
            class="round-dot"
            class:done={status === "done"}
            class:current={status === "current"}
            class:failed={status === "failed"}
            style={status === "current" ? `background: {tileColor};` : ""}
        ></div>
    {/each}
</div>

<p
    class="phase-label"
    class:urgent={phase === "clicking"}
    class:failed={phase === "failed"}
    class:success={phase === "done"}
>
    {phaseLabel}&nbsp;
</p>

<style>
    .controls {
        display: flex;
        align-items: center;
        gap: 1.5rem;
        justify-content: center;
        flex-wrap: wrap;
        margin: 2rem auto 1rem;
    }

    .divider {
        width: 1px;
        height: 2rem;
        background: #3a3a50;
        align-self: center;
    }

    label {
        display: flex;
        flex-direction: column;
        gap: 4px;
        font-size: 0.8rem;
        color: #a0a0b0;
        letter-spacing: 0.05em;
        text-transform: uppercase;
    }

    input,
    select {
        width: 5rem;
        padding: 0.4rem 0.6rem;
        border-radius: 6px;
        border: 1px solid #3a3a50;
        background: #2a2a3e;
        color: #e0e0f0;
        font-size: 1rem;
        text-align: center;
        transition: border-color 150ms;
    }

    select {
        width: auto;
        min-width: 8rem;
        text-align: left;
        cursor: pointer;
    }

    input:focus,
    select:focus {
        outline: none;
        border-color: #7c3aed;
    }

    input:disabled,
    select:disabled {
        opacity: 0.45;
        cursor: not-allowed;
    }

    button {
        padding: 0.5rem 1.5rem;
        border-radius: 6px;
        border: none;
        background: #7c3aed;
        color: white;
        font-size: 1rem;
        cursor: pointer;
        align-self: flex-end;
        margin-bottom: 1px;
        transition:
            background 150ms,
            opacity 150ms;
    }

    button:hover:not(:disabled) {
        background: #6d28d9;
    }
    button:disabled {
        opacity: 0.45;
        cursor: not-allowed;
    }

    .progress-track {
        width: min(90vw, 500px);
        height: 6px;
        background: #2a2a3e;
        border-radius: 3px;
        margin: 0 auto 10px;
        overflow: hidden;
    }

    .progress-bar {
        height: 100%;
        width: 100%;
        border-radius: 3px;
        transform-origin: left;
        transition: background 300ms;
    }

    .color-input {
        display: flex;
        gap: 6px;
        align-items: center;
    }

    .color-input input[type="color"] {
        width: 2rem;
        height: 2rem;
        padding: 2px;
        border-radius: 6px;
        border: 1px solid #3a3a50;
        background: #2a2a3e;
        cursor: pointer;
    }

    input.hex {
        width: 5.5rem;
        font-family: monospace;
        letter-spacing: 0.05em;
    }

    .phase-label {
        text-align: center;
        font-size: 1.1rem;
        color: #a0a0b0;
        margin: 0.5rem 0 1rem;
        min-height: 1.5rem;
        transition: color 200ms;
    }

    .phase-label.urgent {
        color: #e0b0ff;
    }
    .phase-label.failed {
        color: #ef4444;
    }
    .phase-label.success {
        color: #4ade80;
    }

    .grid {
        display: grid;
        grid-template-columns: repeat(var(--cols), 1fr);
        gap: 10px;
        width: min(90vw, 500px);
        margin: 0 auto;
    }

    .round-tracker {
        display: flex;
        gap: 8px;
        justify-content: center;
        margin: 14px auto 4px;
    }

    .round-dot {
        width: 12px;
        height: 12px;
        border-radius: 3px;
        background: #2a2a3e;
        border: 1px solid #3a3a50;
        transition:
            background 300ms,
            border-color 300ms;
    }

    .round-dot.done {
        background: #4ade80;
        border-color: #4ade80;
    }

    .round-dot.failed {
        background: #ef4444;
        border-color: #ef4444;
    }

    .round-dot.current {
        border-color: transparent;
        background-color: var(--tile-color);
    }
</style>
