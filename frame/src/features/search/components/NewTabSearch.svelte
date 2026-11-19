<script lang="ts">
  import { onMount, untrack } from "svelte";
  import { fade } from "svelte/transition";
  import { reducedMotion } from "$shared/lib/motion";
  import { Cancel01Icon } from "@hugeicons/core-free-icons";
  import type { IconSvgElement } from "@hugeicons/svelte";
  import * as m from "$shared/i18n/messages";
  import SearchField from "$shared/ui/SearchField";
  import IconButton from "$shared/ui/IconButton";
  import Icon from "$shared/ui/Icon";
  import ResultList from "./ResultList.svelte";
  import EngineMenu from "./EngineMenu.svelte";
  import { createSearchSurface } from "../lib/search-surface.svelte";
  import { parkCaret } from "$shared/lib/parked-caret";

  let {
    tabId,
    loadMenu,
    onsettings,
    onactive,
    onopen,
    tools = [],
  }: {
    onsettings?: () => void;
    tabId: string | null;
    loadMenu?: () => Promise<typeof import("$shared/ui/Menu")>;
    /** Whether a query is in the field, so the page can make room for it. */
    onactive?: (active: boolean) => void;
    /** Opens one of the tools the bubbles beside the field stand for. */
    onopen?: (tool: "ai" | "downloads" | "bookmarks") => void;
    /** The tools the bubbles stand for, named and drawn by the page that hosts the field. */
    tools?: readonly {
      tool: "ai" | "downloads" | "bookmarks";
      label: string;
      icon: IconSvgElement;
    }[];
  } = $props();

  // The search session is bound to one blank tab for the life of this
  // component; Shell keys the surface on the active tab, so a change is a
  // remount, never a reassignment underneath a live session.
  const surface = createSearchSurface({
    tabId: untrack(() => tabId),
    destinations: [],
    onTool: () => {},
  });

  let input = $state<HTMLInputElement>();
  let root: HTMLDivElement;
  let list = $state<HTMLElement>();
  let content = $state<HTMLElement>();
  let contentHeight = $state(0);
  /** Height is only animated once a first measurement exists, so the sheet
   *  does not grow out of nothing when it opens. */
  let measured = $state(false);
  // The sheet drops into whatever is below the field and never past it.
  let available = $state(420);

  let active = $derived(surface.query.trim().length > 0);
  let focused = $state(false);
  // A parked caret stands in for the focus, so the field keeps its awake look.
  let parked = $state(false);

  // The bubbles bud off the field a moment after it has arrived, and are
  // taken back into it as soon as something is typed.
  let budded = $state(false);
  let split = $derived(budded && !active);
  const BUD_DELAY_MS = 260;

  function picked(next: string | null) {
    if (next !== null) surface.changed(next);
    const field = input;
    if (!field) return;
    field.focus();
    if (next !== null) {
      field.value = next;
      field.setSelectionRange(next.length, next.length);
    }
  }
  let visible = $derived(
    surface.open &&
      active &&
      (surface.rows.length > 0 || surface.empty || surface.failed || surface.error !== "none"),
  );

  $effect(() => {
    surface.setList(list);
  });
  $effect(() => {
    const report = onactive;
    const value = active;
    untrack(() => report?.(value));
  });
  $effect(() => () => untrack(() => onactive)?.(false));
  $effect(() => {
    // Reading the completion is what makes this run on each settled answer.
    void surface.query;
    surface.applyCompletion(input);
  });
  $effect(() => {
    const element = content;
    if (!element) {
      measured = false;
      return;
    }
    // Sizing the sheet to its content turns a result arriving into a single
    // settling movement rather than a jump.
    const observer = new ResizeObserver(([entry]) => {
      // Border box, because the padding is part of what the sheet must be
      // tall enough to show; a content-box measurement leaves a scrollbar.
      contentHeight = entry!.borderBoxSize[0]!.blockSize;
      requestAnimationFrame(() => (measured = true));
    });
    observer.observe(element);
    return () => observer.disconnect();
  });

  function measure() {
    available = Math.max(
      160,
      Math.min(520, window.innerHeight - root.getBoundingClientRect().bottom - 32),
    );
  }

  function keydown(event: KeyboardEvent) {
    // See LauncherPanel: WebKit's committing keydown arrives after compositionend.
    if (surface.composing || event.isComposing || event.keyCode === 229) return;
    if (event.key === "Backspace" && surface.backspace()) {
      event.preventDefault();
    } else if (event.key === "ArrowDown" || event.key === "ArrowUp") {
      event.preventDefault();
      void surface.move(event.key === "ArrowDown" ? 1 : -1);
    } else if (event.key === "Escape") {
      if (surface.dismiss()) event.preventDefault();
    } else if (event.key === "Enter") {
      event.preventDefault();
      surface.submit();
    }
  }

  onMount(() => {
    input?.focus();
    measure();
    window.addEventListener("resize", measure);
    const stop = surface.mount();
    const unpark = input ? parkCaret(input, (value) => (parked = value)) : () => {};
    const bud = setTimeout(() => (budded = true), reducedMotion() ? 0 : BUD_DELAY_MS);
    return () => {
      clearTimeout(bud);
      window.removeEventListener("resize", measure);
      unpark();
      stop();
    };
  });
</script>

<div
  bind:this={root}
  class="dock"
  data-shortcut={surface.activeShortcut?.id}
  data-active={active}
  data-open={visible}
  data-split={split}
  style:--available={`${available}px`}
  style:--count={tools.length}
  onfocusout={(event) => {
    if (event.relatedTarget instanceof Node && root.contains(event.relatedTarget)) return;
    surface.blurred();
  }}
>
  <!-- One sheet of frosted glass: the field is its top, and what is found
       opens out of it. The bubbles beside it are made of the same glass. -->
  <div class="card">
    <div class="bar" data-glass-text>
      <SearchField
        size="page"
        label={surface.activeShortcut
          ? m.search_shortcut_label({ name: surface.activeShortcut.name })
          : m.search_web()}
        placeholder={m.search_web_short()}
        value={surface.query}
        bind:ref={input}
        oninput={surface.changed}
        onsubmit={() => surface.submit()}
        inputProps={{
          role: "combobox",
          "aria-controls": "newtab-search-results",
          "aria-expanded": visible,
          "aria-autocomplete": "both",
          "aria-activedescendant":
            visible && surface.selectedIndex >= 0
              ? `newtab-search-results-option-${surface.selectedIndex}`
              : undefined,
          maxlength: 2048,
          spellcheck: false,
          autocapitalize: "off",
          autocorrect: "off",
          // Room for the clear control, which is placed over the field rather
          // than threaded through the shared primitive for one caller's sake.
          style: surface.query ? "padding-inline-end: 40px" : undefined,
          onkeydown: keydown,
          onfocus: () => {
            focused = true;
            measure();
            surface.focused();
          },
          onblur: () => (focused = false),
          oncompositionstart: surface.compositionStart,
          oncompositionend: (event) => surface.compositionEnd(event.currentTarget.value),
        }}
      >
        {#snippet lead()}<EngineMenu
            query={surface.query}
            awake={focused || parked || active}
            {loadMenu}
            {onsettings}
            onpick={picked}
          />{/snippet}
      </SearchField>
      {#if surface.query}
        <span class="clear">
          <IconButton
            icon={Cancel01Icon}
            label={m.panel_clear_search()}
            onclick={() => {
              surface.changed("");
              input?.focus();
            }}
          />
        </span>
      {/if}
    </div>

    {#if visible}
      <div
        class="sheet"
        data-measured={measured}
        style:block-size={contentHeight ? `min(${contentHeight}px, var(--available))` : undefined}
        out:fade={{ duration: reducedMotion() ? 0 : 90 }}
      >
        <div class="scroll">
          <div class="content" bind:this={content}>
            {#if surface.failed || surface.error !== "none"}
              <div class="failure" role="alert">
                {surface.error === "too_long" ? m.panel_query_too_long() : m.panel_action_failed()}
                <button type="button" onclick={surface.retry}>{m.panel_retry()}</button>
              </div>
            {/if}
            {#if surface.empty}<p class="empty">{m.panel_no_results()}</p>{/if}
            <ResultList
              rows={surface.rows}
              selected={surface.selectedId}
              query={surface.answered}
              listId="newtab-search-results"
              busy={surface.pending}
              variant="launcher"
              bind:ref={list}
              onhover={surface.hover}
              onrun={surface.activate}
            />
          </div>
        </div>
      </div>
    {/if}
  </div>

  <div class="bubbles" inert={!split}>
    {#each tools as bubble, index (bubble.tool)}
      <button
        type="button"
        class="bubble"
        style:--i={index}
        title={bubble.label}
        aria-label={bubble.label}
        onpointerdown={(event) => event.preventDefault()}
        onclick={() => onopen?.(bubble.tool)}
      >
        <Icon icon={bubble.icon} size={19} />
      </button>
    {/each}
  </div>
</div>

<style>
  /* The field floats on the page. It is as wide as the group it belongs to:
     the glass card, and the bubbles that bud off its end. */
  .dock {
    --bar: 46px;
    --bubble: 46px;
    --bubble-gap: 8px;
    --lead-gap: 10px;
    --tray: calc(
      var(--count) * var(--bubble) + (var(--count) - 1) * var(--bubble-gap) + var(--lead-gap)
    );
    --glass: color-mix(in srgb, var(--color-chrome) 52%, transparent);
    --glass-blur: blur(30px) saturate(1.7);
    --glass-shadow: 0 14px 38px color-mix(in srgb, black 26%, transparent);
    --spring: var(--ease-spring);

    position: relative;
    block-size: var(--bar);
    text-align: start;
  }

  /* No border and no ring: the glass is told from the page by its blur and
     its shadow alone. */
  .card {
    position: absolute;
    inset-block-start: 0;
    inset-inline-start: 0;
    inline-size: 100%;
    overflow: hidden;
    border-radius: var(--radius-panel);
    background: var(--glass);
    box-shadow: var(--glass-shadow);
    backdrop-filter: var(--glass-blur);
    transform-origin: 50% 0;
    animation: card-in 170ms cubic-bezier(0.2, 0.8, 0.2, 1) backwards;
    transition: inline-size 460ms var(--spring);
  }

  /* The field gives up the room the bubbles take, and has it back the moment
     something is typed. */
  .dock[data-split="true"] .card {
    inline-size: calc(100% - var(--tray));
  }

  .bar {
    position: relative;
    inline-size: 100%;
  }

  /* The card is the field's shape and material, so the field inside it is
     only its words. Set here rather than as a size of the shared field, which
     every page would pay for on behalf of this one. */
  .dock .bar :global(.ui-search[data-size]),
  .dock .bar :global(.ui-search[data-size]:hover),
  .dock .bar :global(.ui-search[data-size]:focus-within) {
    min-height: var(--bar);
    padding-inline: 18px;
    gap: 11px;
    border: 0;
    border-radius: 0;
    background: transparent;
    box-shadow: none;
    outline: none;
  }

  .dock .bar :global(.ui-search[data-size] > svg) {
    inline-size: 17px;
    block-size: 17px;
  }

  .dock .bar :global(.ui-search[data-size] input) {
    height: var(--bar);
    font-size: 16px;
    letter-spacing: -0.01em;
    outline: none;
  }

  .clear {
    position: absolute;
    inset-block: 0;
    inset-inline-end: 10px;
    display: flex;
    align-items: center;
  }

  /* What is found opens out of the field, as one piece of the same glass. */
  .sheet {
    --row-radius: calc(var(--radius-panel) - 6px);

    max-block-size: var(--available);
    overflow: hidden;
    animation: sheet-in 220ms var(--ease-out) backwards;
  }

  /* The height goes straight for each answer as it lands, longer or
     shorter, in one long settle that picks up from wherever it is when the
     next answer comes mid-way. */
  .sheet[data-measured="true"] {
    transition: block-size var(--motion-slow) var(--ease-emphasized);
  }

  .scroll {
    max-block-size: var(--available);
    overflow-y: auto;
    overscroll-behavior: contain;
  }

  /* Padding lives on the measured element so the animated height accounts for
     it. */
  .content {
    padding: 2px 6px 6px;
  }

  .empty {
    margin: 0;
    padding: 22px 12px;
    color: var(--color-faint);
    font-size: 13px;
    text-align: center;
  }

  .failure {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding: 8px 10px;
    color: var(--color-danger);
    font-size: 12px;
  }

  .failure button {
    padding: 5px 10px;
    border: 0;
    border-radius: var(--radius-inset);
    background: var(--color-fill);
    color: var(--color-text);
    cursor: default;
  }

  /* The bubbles wait where the field's end will be, small and unseen. When
     the field draws back they spring out of it one after another and settle
     in a row; they are drawn back in the other way. */
  .bubbles {
    position: absolute;
    inset-block-start: 0;
    inset-inline-end: 0;
    display: flex;
    gap: var(--bubble-gap);
    pointer-events: none;
  }

  .bubble {
    --from: calc(var(--i) * (var(--bubble) + var(--bubble-gap)) * -1);

    display: grid;
    place-items: center;
    inline-size: var(--bubble);
    block-size: var(--bubble);
    padding: 0;
    border: 0;
    border-radius: var(--radius-capsule);
    background: var(--glass);
    box-shadow: var(--glass-shadow);
    backdrop-filter: var(--glass-blur);
    color: var(--color-text);
    opacity: 0;
    transform: translateX(var(--from)) scale(0.25);
    pointer-events: none;
    transition:
      transform 460ms var(--spring) calc((var(--count) - var(--i) - 1) * 35ms),
      opacity 160ms var(--ease-out) calc((var(--count) - var(--i) - 1) * 35ms),
      background-color var(--motion-fast) var(--ease-out);
  }

  .bubble:focus-visible {
    outline: 2px solid var(--color-accent);
    outline-offset: 2px;
  }

  :global(:root[data-reduce-motion="true"]) .card,
  :global(:root[data-reduce-motion="true"]) .sheet,
  :global(:root[data-reduce-motion="true"]) .bubble {
    animation: none;
    transition: none;
  }

  .dock[data-split="true"] .bubble {
    opacity: 1;
    transform: none;
    pointer-events: auto;
    transition-delay: calc(var(--i) * 45ms), calc(var(--i) * 45ms), 0ms;
  }

  .dock[data-split="true"] .bubble:hover {
    background: color-mix(in srgb, var(--color-chrome) 52%, var(--color-text) 14%);
  }

  .dock[data-split="true"] .bubble:active {
    transform: scale(0.94);
    transition-duration: var(--motion-fast);
    transition-delay: 0ms;
  }

  @keyframes card-in {
    from {
      opacity: 0;
      transform: scale(0.9);
    }
  }

  @keyframes sheet-in {
    from {
      opacity: 0;
      transform: translateY(-6px);
    }
  }

  @media (prefers-reduced-motion: reduce) {
    .card,
    .sheet,
    .bubble {
      animation: none;
      transition: none;
    }
  }

  @media (forced-colors: active) {
    .card,
    .bubble {
      border: 1px solid CanvasText;
    }
  }
</style>
