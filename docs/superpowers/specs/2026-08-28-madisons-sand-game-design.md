# Madison's Sand Game — Design

**Date:** 2026-08-28
**Upstream:** forked from [charlesguse/sand-game](https://github.com/charlesguse/sand-game)
**Fork:** [m8j8k/madisons-sand-game](https://github.com/m8j8k/madisons-sand-game)

## Who this is for

Madison, almost 5. She likes pink sand, sparkles, rainbows, unicorns, pink
water, poodles, gumdrops, and palm trees. She plays on an iPad.

Guse built the original for his own almost-5-year-old, and her tastes overlap
Madison's almost exactly. That is why this is a fork and not a rewrite: the
engine, the sim rules, and roughly 4,600 lines of passing tests already do
most of what Madison wants. Our delta is small and specific.

## What we are changing

| Change | Kind |
|---|---|
| iPad gesture containment + fullscreen | Fix (urgent) |
| Constitution rewritten for Madison | Governance |
| Water recolored blue → pink | Tweak |
| Gumdrops | New element |
| Poodles | New entity type |
| Swaying palm trees | New object kind |

Everything else stays as guse built it.

## Non-negotiables we inherit

These come from the upstream constitution and stay in force. They are good
engineering constraints regardless of whose child is playing.

- **Single self-contained `index.html`.** No external requests at runtime; must
  work from `file://`. This rules out a PWA manifest file (see iPad section).
- **Playable without reading.** Big emoji-labeled buttons only.
- **No failure states.** No scores, no timers, nothing she can do wrong.
- **60fps target, ≥30fps floor.** Hot loop stays allocation-free.
- **No browser test harness.** CI has no browser; sim rules are covered by
  plain vitest on grid logic.

## Phase 0 — the iPad fix (urgent, ships outside the pipeline)

Madison is being thrown out of the game mid-play because her drags trigger
iPadOS system gestures. This is breaking her *today*, so it ships directly
rather than waiting on pipeline setup.

**Revised 2026-08-28 after reading upstream source.** The original draft of this
section listed in-page gesture containment work that upstream has *already
done* in its "Phone Support (#15)" feature. Verified present:

- `touch-action: none` on `html`/`body` and on `.play-area` itself
- `overscroll-behavior: none`, `user-select: none`, `-webkit-touch-callout: none`
- `viewport-fit=cover`, `user-scalable=no`, `maximum-scale=1.0`
- `env(safe-area-inset-*)` padding on all four sides of the toolbar
- Pointer events with `setPointerCapture`

None of that needs rebuilding. That narrows the diagnosis: the page is already
doing everything a page can do, so what remains is Safari's own chrome and
iPadOS system gestures.

**What is actually missing, and is what Phase 0 builds:**

1. **`<meta name="apple-mobile-web-app-capable" content="yes">`** plus a status
   bar style. Without it, **Add to Home Screen** still launches inside Safari
   with its chrome and its edge behaviors. With it, the game launches
   standalone — no address bar, no tab strip, and materially fewer ways for a
   4-year-old to fall out of it. This meta tag needs no manifest file, which is
   what keeps it inside the single-file principle.
2. **A fullscreen button.** A large ⛶ control calling `requestFullscreen()`
   (with the `webkit` prefixed fallback older iPadOS Safari needs). It must
   **hide itself when unsupported** — a dead button violates the no-failure-
   states principle.
3. **Guided Access documentation.** The part code cannot do.

**What code cannot fix — iPadOS system edge gestures.** No web page can block
the app switcher, Control Center, or the home-indicator swipe. The real answer
is **Guided Access**: Settings → Accessibility → Guided Access, then
triple-click the side button to lock the iPad into the game. It can also
disable chosen screen regions outright. This goes in the README so it is not
tribal knowledge, and it is the single highest-value thing Max can do for
Madison's experience today.

## Phase 1 — make it hers

Rewrite `.specify/memory/constitution.md` for Madison. This file steers every
agent in the Wing Commander pipeline, so it must be correct **before** any
feature runs through. Engineering principles carry over verbatim; the product
section becomes hers.

Recolor `WATER` from blue to pink. We recolor the existing element rather than
adding a second liquid — the constitution says keep the element set small, and
grass should still drink it.

Retitle to Madison's game.

## Phase 2 — Wing Commander

The pipeline is seven chained workflows calling
`charlesguse/wing-commander/.github/workflows/*@main`. A `spec-request` label
applied by the maintainer is the security gate that starts intake.

Setup required:

1. **Enable Actions on the fork** — disabled by default on all forks
2. **Create the `spec-request` label**
3. **Register a GitHub App** and supply `WING_COMMANDER_APP_ID` and
   `WING_COMMANDER_APP_PRIVATE_KEY`
4. **Supply `CLAUDE_CODE_OAUTH_TOKEN` and `ANTHROPIC_API_KEY`**
5. Optionally set `WING_COMMANDER_MAX_ITERATIONS`, `WING_COMMANDER_PLAN_REVIEW`,
   `WING_COMMANDER_TASKS_REVIEW`

Secrets are provisioned by Max directly through the GitHub UI or `gh secret
set`. Claude does not handle their values.

Prove the pipeline with one trivial issue before routing real features through
it.

## Phase 3 — Gumdrops

New element `GUMDROP`. Falls under gravity but does **not** flow flat the way
sand does — high friction, so it settles into chunky candy heaps. Each gumdrop
picks a random candy hue from the existing `hues` array, so a poured handful
comes out multicolored.

Explicitly out of scope, considered and cut: dissolving in water, bouncing
physics, and gumdrop rain. Gumdrops exist to be piled up and chased.

## Phase 4 — Palm trees

A third object kind alongside rainbow and unicorn, glyph `'🌴'`.

Objects already stamp an `OBJECT` footprint into the grid, so **sand piles
against the trunk and water pools around it with no new sim code**.

The sway is purely a render-layer effect in `PlayArea.svelte`: translate to the
base of the trunk, `ctx.rotate(Math.sin(now * speed + phase) * amplitude)`,
draw the glyph, restore. Two details carry it — pivot at the **base** so it
bends like a trunk instead of spinning like a pinwheel, and give each tree its
own **phase offset** so a row ripples rather than swaying in lockstep. This
matches the existing `Math.sin` shimmer pattern already in the render loop.

**Known approximation:** the drawn glyph sways while its collision footprint
stays put. At a few degrees of gentle sway this reads as breeze, not as a bug.
Accepted deliberately.

Deferred as YAGNI: a shared global wind value driving both palm sway and
fog/rain drift. Nice, not needed.

## Phase 5 — The poodle

The centerpiece, and deliberately **not** a grid element. Moving creatures in a
cellular automaton are miserable — double-stepping, moved-flag bookkeeping,
partial updates. Instead:

A new `src/sim/pets.ts` owns a small list of poodles. Each has a position, a
facing, a state, a target, and a timer. It is stepped once per frame *after*
the grid step, reading the grid to find the ground surface and writing to it to
groom, dig, and eat. There are only ever a handful, so per-frame cost is
negligible and the performance principle is safe. It is pure logic with no DOM,
so it is unit-testable exactly the way the constitution requires.

**Behavior loop, in priority order:**

1. **Eating** — a gumdrop within range beats everything else; on arrival the
   gumdrop is consumed and the poodle does a sparkle wiggle
2. **Shaking** — if it just crossed water it is soggy; it stops and sprays pink
   droplets
3. **Digging** — if buried under sand, it digs out, flinging sand behind it
4. **Trotting** — otherwise it walks toward wherever Madison last touched
5. **Idle** — with no target, it wanders gently

**Grooming** happens passively while trotting: plain sand it walks over becomes
rainbow sand, and it leaves sparkle pawprints.

The design consequence worth naming: dropping a gumdrop becomes Madison's way
to *steer the dog*. That is what makes the poodle and the gumdrops one toy
rather than two unrelated buttons.

Shipped in that order as 5a through 5e — each step is independently playable,
so she gets something new every time. 5a (trot toward her finger, sparkle
pawprints) is the smallest thing worth shipping; the rest layer on top without
reworking it.

## Code health work, scoped to what we are touching

Two upstream patterns will actively fight the features above. Both are fixed as
part of the work, and neither is speculative refactoring.

**`moveCell` / `swapCells` in `src/sim/step.ts`** hand-copy 13 parallel typed
arrays field by field. Adding `GUMDROP` means editing both again, and omitting
one line is a silent corruption bug that tests only catch if someone thought to
write that exact case. Replace with a single field list driving both functions.

**`ObjectsState` in `src/sim/objects.ts`** is hardcoded as
`{ rainbows, unicorns }`, with a ternary in `listFor()` and a two-loop
`isCoveredByAnyObject`. Adding palms touches all of it. A record keyed by
`ObjectKind` makes this and any future kind a one-line change.

## How this decomposes

This document is a **program design, not a single implementation plan**. Each
phase is independently shippable and gets its own plan:

| Phase | Scope | Route |
|---|---|---|
| 0 | Standalone meta, fullscreen button, Guided Access docs | Direct — urgent |
| 1 | Constitution, pink water, retitle | Direct — precedes the pipeline |
| 2 | Wing Commander setup | Direct — Max provisions secrets |
| 3 | Gumdrops | Pipeline |
| 4 | Palm trees | Pipeline |
| 5a–5e | Poodle behavior loop | Pipeline, one issue per behavior |

Phases 0–2 run directly because they are either urgent or are the thing that
makes the pipeline exist. Everything from phase 3 on goes through Wing
Commander as a labeled issue, which is the point of the fork.

The two code-health items are not their own phase. Each rides along with the
first feature that touches it: the `moveCell` field list lands with gumdrops,
the keyed `ObjectsState` lands with palm trees.

## Testing

Per the constitution: plain vitest, no DOM, no browser automation.

- `pets.ts` — construct a grid, place a poodle, step it, assert position and
  state transitions. Every branch of the behavior loop gets a case.
- Gumdrop settling — assert heaps stay chunky and do not flatten like sand.
- Palm placement — assert the footprint stamps and that the keyed
  `ObjectsState` refactor preserves existing rainbow/unicorn behavior.
- The `moveCell` field-list refactor is covered by the existing suite; it must
  stay green with no test edits.

Sway animation, pink water, and the iPad fixes are visual or device-level and
are checked by eye — the constitution explicitly assigns that to review time.

## Licensing note

Upstream has no LICENSE file, so it is formally all-rights-reserved. Forking
within GitHub is fine under their terms, but we will be deploying our own Pages
copy. Max is telling guse directly. Not a blocker between friends, but worth
recording.
