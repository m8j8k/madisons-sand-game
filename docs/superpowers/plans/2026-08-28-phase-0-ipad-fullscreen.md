# Phase 0 — iPad Standalone & Fullscreen Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop Madison from being thrown out of the game on her iPad, by making it launch standalone from the Home Screen, adding a fullscreen control, and documenting Guided Access.

**Architecture:** Two meta tags in `index.html` enable iOS standalone launch. A new dependency-injected `src/lib/fullscreen.ts` module wraps the prefixed Fullscreen APIs so it is unit-testable with plain objects and no DOM. `Toolbar.svelte` gains one button wired through `App.svelte`, following the existing `onClearAll` prop pattern exactly.

**Tech Stack:** Svelte 5 (runes: `$state`, `$props`), TypeScript, Vite + `vite-plugin-singlefile`, vitest (node environment, no DOM).

## Global Constraints

Copied verbatim from `.specify/memory/constitution.md`. Every task's requirements implicitly include these.

- **Single self-contained `index.html`.** No external network requests at runtime. Must work from `file://`. No manifest file, no external assets.
- **Playable without reading.** Controls are big, colorful, emoji-labeled buttons.
- **No failure states.** Nothing she does is ever "wrong". A control that does nothing when pressed is a failure state — hide it instead.
- **Works with mouse *and* touch.**
- **Performance:** 60fps target, ≥30fps floor. Hot loop stays allocation-free. (Phase 0 adds nothing to the hot loop.)
- **Verifiable without a browser harness.** CI has no browser. Cover logic with plain vitest. **Do not add browser-automation test infrastructure.**
- Touch targets respect `MIN_TOUCH_TARGET` from `src/lib/layout.ts`, already applied via the `--control-min` CSS variable on `.toolbar`.

## Context: what already exists

Upstream already ships all in-page gesture containment. **Do not re-add any of this** — it is present and working:

- `touch-action: none` on `html`/`body` (`index.html`) and on `.play-area` (`src/lib/PlayArea.svelte`)
- `overscroll-behavior: none`, `user-select: none`, `-webkit-user-select: none`, `-webkit-touch-callout: none`
- `viewport-fit=cover`, `user-scalable=no`, `maximum-scale=1.0`
- `env(safe-area-inset-*)` padding on all four sides of `.toolbar`
- Pointer events with `setPointerCapture`

---

### Task 1: iOS standalone meta tags

Makes **Add to Home Screen** launch without Safari chrome.

**Files:**
- Modify: `index.html` (head, after the existing `theme-color` meta)
- Test: `tests/unit/shell/indexHtml.test.ts` (create)

**Interfaces:**
- Consumes: nothing
- Produces: nothing consumed by later tasks (pure HTML change)

- [ ] **Step 1: Write the failing test**

Create `tests/unit/shell/indexHtml.test.ts`:

```ts
import { readFileSync } from 'node:fs';
import { fileURLToPath } from 'node:url';
import { describe, it, expect } from 'vitest';

const html = readFileSync(
  fileURLToPath(new URL('../../../index.html', import.meta.url)),
  'utf8',
);

describe('index.html iOS standalone shell', () => {
  it('declares itself an iOS standalone web app', () => {
    expect(html).toContain('name="apple-mobile-web-app-capable"');
    expect(html).toMatch(
      /name="apple-mobile-web-app-capable"[^>]*content="yes"/,
    );
  });

  it('sets a status bar style that lets the page paint under the bar', () => {
    expect(html).toMatch(
      /name="apple-mobile-web-app-status-bar-style"[^>]*content="black-translucent"/,
    );
  });

  it('keeps viewport-fit=cover so safe-area insets resolve', () => {
    expect(html).toContain('viewport-fit=cover');
  });

  it('makes no external network requests', () => {
    expect(html).not.toMatch(/(?:src|href)="https?:\/\//);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run tests/unit/shell/indexHtml.test.ts`
Expected: FAIL — the first two assertions fail because the meta tags are absent. The `viewport-fit` and external-request tests should already pass.

- [ ] **Step 3: Write minimal implementation**

In `index.html`, immediately after the existing line:

```html
    <meta name="theme-color" content="#ffe1f0" />
```

add:

```html
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
    <meta name="apple-mobile-web-app-title" content="Madison's Sand" />
```

`black-translucent` is deliberate: it lets the page paint the full height, and the toolbar's existing `env(safe-area-inset-top)` padding already keeps controls clear of the status bar.

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run tests/unit/shell/indexHtml.test.ts`
Expected: PASS, 4 tests.

- [ ] **Step 5: Commit**

```bash
git add index.html tests/unit/shell/indexHtml.test.ts
git commit -m "feat: launch standalone from iPad Home Screen"
```

---

### Task 2: Fullscreen module

Pure logic, dependency-injected so it tests without a DOM.

**Files:**
- Create: `src/lib/fullscreen.ts`
- Test: `tests/unit/lib/fullscreen.test.ts` (create)

**Interfaces:**
- Consumes: nothing
- Produces — Task 3 relies on these exact signatures:
  - `export interface FullscreenElement { requestFullscreen?: () => Promise<void>; webkitRequestFullscreen?: () => Promise<void> | void; }`
  - `export interface FullscreenDocument { fullscreenElement?: Element | null; webkitFullscreenElement?: Element | null; exitFullscreen?: () => Promise<void>; webkitExitFullscreen?: () => Promise<void> | void; }`
  - `export function isFullscreenSupported(element: FullscreenElement): boolean`
  - `export function isFullscreen(doc: FullscreenDocument): boolean`
  - `export function toggleFullscreen(element: FullscreenElement, doc: FullscreenDocument): Promise<void>`

- [ ] **Step 1: Write the failing test**

Create `tests/unit/lib/fullscreen.test.ts`:

```ts
import { describe, it, expect, vi } from 'vitest';
import {
  isFullscreenSupported,
  isFullscreen,
  toggleFullscreen,
  type FullscreenElement,
  type FullscreenDocument,
} from '../../../src/lib/fullscreen';

describe('isFullscreenSupported', () => {
  it('is true when the standard API exists', () => {
    expect(isFullscreenSupported({ requestFullscreen: async () => {} })).toBe(true);
  });

  it('is true when only the webkit API exists (older iPadOS Safari)', () => {
    expect(isFullscreenSupported({ webkitRequestFullscreen: () => {} })).toBe(true);
  });

  it('is false when neither exists (iPhone Safari)', () => {
    expect(isFullscreenSupported({})).toBe(false);
  });
});

describe('isFullscreen', () => {
  it('is false when no element is fullscreen', () => {
    expect(isFullscreen({ fullscreenElement: null })).toBe(false);
  });

  it('is true via the standard property', () => {
    expect(isFullscreen({ fullscreenElement: {} as Element })).toBe(true);
  });

  it('is true via the webkit property', () => {
    expect(isFullscreen({ webkitFullscreenElement: {} as Element })).toBe(true);
  });

  it('is false for an empty document object', () => {
    expect(isFullscreen({})).toBe(false);
  });
});

describe('toggleFullscreen', () => {
  it('requests fullscreen when not currently fullscreen', async () => {
    const requestFullscreen = vi.fn(async () => {});
    const element: FullscreenElement = { requestFullscreen };
    const doc: FullscreenDocument = { fullscreenElement: null };

    await toggleFullscreen(element, doc);

    expect(requestFullscreen).toHaveBeenCalledOnce();
  });

  it('prefers the webkit request when the standard one is absent', async () => {
    const webkitRequestFullscreen = vi.fn(() => {});
    const element: FullscreenElement = { webkitRequestFullscreen };

    await toggleFullscreen(element, {});

    expect(webkitRequestFullscreen).toHaveBeenCalledOnce();
  });

  it('exits fullscreen when already fullscreen', async () => {
    const exitFullscreen = vi.fn(async () => {});
    const doc: FullscreenDocument = { fullscreenElement: {} as Element, exitFullscreen };

    await toggleFullscreen({ requestFullscreen: async () => {} }, doc);

    expect(exitFullscreen).toHaveBeenCalledOnce();
  });

  it('resolves without throwing when nothing is supported', async () => {
    await expect(toggleFullscreen({}, {})).resolves.toBeUndefined();
  });

  it('swallows a rejected request so the toy never shows an error', async () => {
    const element: FullscreenElement = {
      requestFullscreen: async () => {
        throw new Error('denied');
      },
    };

    await expect(toggleFullscreen(element, {})).resolves.toBeUndefined();
  });
});
```

The last test encodes the no-failure-states principle: a refused fullscreen request must never surface as an error.

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run tests/unit/lib/fullscreen.test.ts`
Expected: FAIL — `Failed to resolve import "../../../src/lib/fullscreen"`.

- [ ] **Step 3: Write minimal implementation**

Create `src/lib/fullscreen.ts`:

```ts
/**
 * Thin wrapper over the Fullscreen API, with the `webkit` fallbacks older
 * iPadOS Safari still needs. Dependency-injected rather than reaching for
 * globals so it unit-tests with plain objects and no DOM (constitution V).
 */

export interface FullscreenElement {
  requestFullscreen?: () => Promise<void>;
  webkitRequestFullscreen?: () => Promise<void> | void;
}

export interface FullscreenDocument {
  fullscreenElement?: Element | null;
  webkitFullscreenElement?: Element | null;
  exitFullscreen?: () => Promise<void>;
  webkitExitFullscreen?: () => Promise<void> | void;
}

export function isFullscreenSupported(element: FullscreenElement): boolean {
  return (
    typeof element.requestFullscreen === 'function' ||
    typeof element.webkitRequestFullscreen === 'function'
  );
}

export function isFullscreen(doc: FullscreenDocument): boolean {
  return Boolean(doc.fullscreenElement ?? doc.webkitFullscreenElement);
}

/**
 * Enters or leaves fullscreen. Never rejects: a browser that refuses the
 * request (or has no API at all) must not surface an error to a 4-year-old.
 */
export async function toggleFullscreen(
  element: FullscreenElement,
  doc: FullscreenDocument,
): Promise<void> {
  try {
    if (isFullscreen(doc)) {
      if (typeof doc.exitFullscreen === 'function') {
        await doc.exitFullscreen();
      } else if (typeof doc.webkitExitFullscreen === 'function') {
        await doc.webkitExitFullscreen();
      }
      return;
    }

    if (typeof element.requestFullscreen === 'function') {
      await element.requestFullscreen();
    } else if (typeof element.webkitRequestFullscreen === 'function') {
      await element.webkitRequestFullscreen();
    }
  } catch {
    // Deliberately ignored — see doc comment.
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run tests/unit/lib/fullscreen.test.ts`
Expected: PASS, 12 tests.

- [ ] **Step 5: Commit**

```bash
git add src/lib/fullscreen.ts tests/unit/lib/fullscreen.test.ts
git commit -m "feat: add dependency-injected fullscreen helper"
```

---

### Task 3: Fullscreen button in the toolbar

**Files:**
- Modify: `src/lib/Toolbar.svelte` (props interface ~lines 5-29; `history` group ~line 116)
- Modify: `src/App.svelte` (script block and `<Toolbar>` usage)
- Test: manual — Svelte component rendering needs a DOM, and the constitution forbids adding browser test infrastructure. Task 2 already covers the logic.

**Interfaces:**
- Consumes: `isFullscreenSupported`, `toggleFullscreen` from `src/lib/fullscreen.ts` (Task 2)
- Produces: a `showFullscreen: boolean` and `onToggleFullscreen: () => void` prop pair on `Toolbar`

- [ ] **Step 1: Add the props to Toolbar's interface**

In `src/lib/Toolbar.svelte`, add to `interface Props` after `canRedo: boolean;`:

```ts
    showFullscreen: boolean;
```

and after `onRedo: () => void;`:

```ts
    onToggleFullscreen: () => void;
```

Then add `showFullscreen,` and `onToggleFullscreen,` to the `$props()` destructuring list, matching the existing order.

- [ ] **Step 2: Add the button**

In `src/lib/Toolbar.svelte`, immediately after the closing `</div>` of the `history` group (the one containing Undo/Redo), add:

```svelte
  {#if showFullscreen}
    <div class="group screen">
      <button class="control" aria-label="Full screen" onclick={onToggleFullscreen}>⛶</button>
    </div>
  {/if}
```

The `{#if}` is the no-failure-states requirement: on a browser without fullscreen support the button is absent rather than inert.

- [ ] **Step 3: Wire it in App.svelte**

In `src/App.svelte`, add to the `<script>` block after the existing imports:

```ts
  import { isFullscreenSupported, toggleFullscreen } from './lib/fullscreen';
```

after `let playArea: PlayArea;` add:

```ts
  const showFullscreen = isFullscreenSupported(document.documentElement);

  function handleToggleFullscreen(): void {
    void toggleFullscreen(document.documentElement, document);
  }
```

and add these two attributes to the `<Toolbar ... />` usage:

```svelte
    {showFullscreen}
    onToggleFullscreen={handleToggleFullscreen}
```

`document.documentElement` is the target so the whole page goes fullscreen, not just the canvas — the toolbar must stay reachable.

- [ ] **Step 4: Verify the build and the full suite**

Run: `npm run build && npm test`
Expected: build succeeds and emits `dist/index.html`; all tests pass. If TypeScript complains that `document.documentElement` does not satisfy `FullscreenElement`, the structural types in Task 2 are optional properties so it should assign cleanly — do **not** add a cast; fix the interface instead.

- [ ] **Step 5: Confirm the single-file principle still holds**

Run: `grep -cE '(src|href)="https?://' dist/index.html`
Expected: `0`.

- [ ] **Step 6: Commit**

```bash
git add src/lib/Toolbar.svelte src/App.svelte
git commit -m "feat: add fullscreen button, hidden where unsupported"
```

---

### Task 4: Guided Access documentation

The highest-value part of this phase, and the part code cannot do.

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: nothing
- Produces: nothing

- [ ] **Step 1: Add the section**

Append to `README.md`:

```markdown
## Setting it up on an iPad

Two steps, and the second one matters more than anything in the code.

**1. Add it to the Home Screen.** Open the game in Safari, tap Share → *Add to
Home Screen*. Launched from that icon it runs standalone — no address bar, no
tab strip. There is also a ⛶ button in the toolbar for fullscreen in a normal
browser tab.

**2. Turn on Guided Access.** This is what actually stops a small child from
falling out of the game. No web page can block the iPad's own edge gestures —
the app switcher, Control Center, and the swipe-up home indicator are system
level and unreachable from a web page. Guided Access is Apple's answer:

- Settings → Accessibility → Guided Access → turn it **on**
- Set a passcode under *Passcode Settings*
- Open the game, then **triple-click the side button** to start a session
- Optionally circle any screen area you want disabled before tapping *Start*
- Triple-click again and enter the passcode to end the session

While a session is running the iPad is locked to the game. That is the setup
that lets you hand her the iPad and walk away.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: iPad Home Screen and Guided Access setup"
```

---

## Verification

Run the full suite and build one final time:

```bash
npm test && npm run build
```

Expected: all tests pass, `dist/index.html` is emitted as a single file with zero external references.

**Maintainer eyeball checks** (the constitution assigns visual verification to review time):

1. Open `dist/index.html` directly from disk via `file://` — the game runs, the ⛶ button appears and toggles fullscreen.
2. On the iPad, Add to Home Screen and launch from the icon — no Safari chrome, toolbar clear of the status bar and home indicator.
3. Start a Guided Access session and confirm edge swipes no longer leave the game.
