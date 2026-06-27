<script>
    /**
     * Tile.svelte  (Svelte 5 runes)
     * A single tile for a memory game.
     *
     * Props:
     *   color       – the highlight color shown when the tile is active/revealed
     *   active      – whether this tile is part of the current round's active set
     *   clickable   – whether the player is allowed to click this tile right now
     *   onReveal    – called when show() is invoked (e.g. to start a reveal animation upstream)
     *   onClick     – called with (active: boolean) when the player clicks a clickable tile
     */

    let {
        color = "#3f8fe9",
        active = false,
        clickable = false,
        onReveal = () => {},
        onClick = (_wasActive) => {},
    } = $props();

    // Internal display state
    let revealed = $state(false); // currently showing color (reveal phase)
    let guessed = $state(false); // player already clicked this tile
    let correct = $state(null); // null | true | false — result feedback

    // Derived visual state label (for ARIA)
    const state = $derived(
        revealed
            ? "revealed"
            : guessed
              ? correct
                  ? "correct"
                  : "wrong"
              : clickable
                ? "clickable"
                : "idle",
    );

    /**
     * show() – call this to flash the tile's highlight color.
     * Typically called by the parent for all active tiles simultaneously.
     * Returns a Promise that resolves when the tile hides itself again.
     */
    export async function show(duration = 1500) {
        revealed = true;
        onReveal();
        await wait(duration);
        revealed = false;
    }

    /** reset() – clear guessed/correct state between rounds */
    export function reset() {
        revealed = false;
        guessed = false;
        correct = null;
    }

    /** reveal() – permanently light up the tile (expose correct answer on failure) */
    export function reveal() {
        revealed = true;
    }

    function handleClick() {
        if (!clickable || guessed) return;
        guessed = true;
        correct = active;
        onClick(active);
    }

    function wait(ms) {
        return new Promise((r) => setTimeout(r, ms));
    }
</script>

<!-- svelte-ignore a11y_click_events_have_key_events -->
<button
    class="tile"
    class:revealed
    class:clickable={clickable && !guessed}
    class:correct={guessed && correct === true}
    class:wrong={guessed && correct === false}
    aria-label="Memory tile, {state}"
    aria-pressed={guessed || undefined}
    disabled={!clickable || guessed}
    onclick={handleClick}
    style="--tile-color: {color};"
>
    <span class="inner" />
</button>

<style>
    .tile {
        /* sizing comes from parent grid; tile fills its cell */
        display: block;
        width: 100%;
        aspect-ratio: 1;
        padding: 0;
        border: none;
        border-radius: 10px;
        cursor: default;
        background: var(--tile-bg, #1e1e2e);
        box-shadow:
            0 2px 6px rgba(0, 0, 0, 0.35),
            inset 0 1px 0 rgba(255, 255, 255, 0.06);
        transition:
            background 180ms ease,
            box-shadow 180ms ease,
            transform 120ms ease;
        position: relative;
        overflow: hidden;
    }

    /* ── subtle sheen overlay ─────────────────────────────────── */
    .inner {
        position: absolute;
        inset: 0;
        border-radius: inherit;
        background: linear-gradient(
            135deg,
            rgba(255, 255, 255, 0.08) 0%,
            transparent 60%
        );
        pointer-events: none;
    }

    /* ── states ───────────────────────────────────────────────── */

    /* reveal phase: show the color */
    .tile.revealed {
        background: var(--tile-color);
        box-shadow:
            0 0 18px 4px var(--tile-color),
            0 2px 6px rgba(0, 0, 0, 0.35);
        transform: scale(1.05);
    }

    /* player can click: subtle pulse border */
    .tile.clickable {
        cursor: pointer;
        box-shadow:
            0 2px 6px rgba(0, 0, 0, 0.35),
            inset 0 1px 0 rgba(255, 255, 255, 0.06),
            0 0 0 2px rgba(255, 255, 255, 0.12);
        animation: idle-pulse 2s ease-in-out infinite;
    }
    .tile.clickable:hover {
        transform: scale(1.04);
        box-shadow:
            0 4px 12px rgba(0, 0, 0, 0.4),
            0 0 0 2px rgba(255, 255, 255, 0.2);
    }
    .tile.clickable:active {
        transform: scale(0.96);
    }

    /* correct guess */
    .tile.correct {
        background: var(--tile-color);
        box-shadow: 0 0 14px 3px var(--tile-color);
        cursor: default;
    }

    /* wrong guess */
    .tile.wrong {
        background: #ef4444;
        box-shadow: 0 0 14px 3px #ef4444;
        animation: shake 320ms ease forwards;
        cursor: default;
    }

    /* ── keyframes ────────────────────────────────────────────── */
    @keyframes idle-pulse {
        0%,
        100% {
            opacity: 1;
        }
        50% {
            opacity: 0.8;
        }
    }

    @keyframes shake {
        0% {
            transform: translateX(0);
        }
        20% {
            transform: translateX(-5px);
        }
        40% {
            transform: translateX(5px);
        }
        60% {
            transform: translateX(-4px);
        }
        80% {
            transform: translateX(4px);
        }
        100% {
            transform: translateX(0);
        }
    }

    /* respect reduced-motion */
    @media (prefers-reduced-motion: reduce) {
        .tile,
        .tile.revealed,
        .tile.clickable {
            transition:
                background 80ms,
                box-shadow 80ms;
            animation: none;
            transform: none !important;
        }
    }
</style>
