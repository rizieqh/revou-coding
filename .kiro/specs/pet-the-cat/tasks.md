# Implementation Plan: Pet the Cat

## Overview

The core app is already implemented in a single `index.html`. These tasks cover verifying and hardening the existing implementation against all requirements, adding the missing correctness checks (pool validation, localStorage safety, reduced-motion, focus indicator), and building the full property-based and integration test suite described in the design document.

All implementation is vanilla JavaScript / HTML / CSS — no build step. Tests use `fast-check` for property-based testing and `Playwright` for integration tests, both driven by a minimal `package.json` test harness.

---

## Tasks

- [x] 1. Audit and patch core correctness gaps in `index.html`
  - [x] 1.1 Add reaction pool validation on IIFE init
    - At the top of the IIFE, after `REACTIONS` is defined, check `REACTIONS.length >= 8`. If the check fails, hide the cat stage and render a visible error banner in its place; do not attach event listeners.
    - _Requirements: 3.5 — Property 7_

  - [x] 1.2 Wrap localStorage access in try/catch
    - Wrap the `localStorage.getItem` init read and the `localStorage.setItem` call in `onPet` inside individual try/catch blocks. On any thrown exception, default `purrCount` to 0 and skip all future `setItem` calls for the session (use a `canPersist` flag).
    - _Requirements: 4.3, 4.5 — Property 9, 10_

  - [x] 1.3 Add `prefers-reduced-motion` CSS override
    - In the `<style>` block, add an `@media (prefers-reduced-motion: reduce)` rule that sets `animation: none !important` and `transition: none !important` on `#cat-svg`, `.float-text`, `.confetti`, `#cat-tail`, and `#counter-badge`. Cat SVG should fall back to a static opacity pulse (CSS `opacity` only, no transform) so interaction still registers visually.
    - _Requirements: 6.7_

  - [x] 1.4 Add visible keyboard focus indicator
    - Add a `:focus-visible` CSS rule on `#cat-wrap` that applies an `outline` of at least `3px solid` with a color that achieves ≥ 3:1 contrast against the `#fdf6ee` background (e.g., `#b45309` amber-700). Remove any `outline: none` overrides.
    - _Requirements: 6.6_

- [x] 2. Verify and complete interaction requirements in `index.html`
  - [x] 2.1 Verify animation timing and interrupt behaviour
    - Confirm `applyReaction` clears `animationTimeout`, forces a reflow, and starts the new animation all within the synchronous call stack of `onPet`. Add an explicit guard: if `catSvg.style.animation` is non-empty when `onPet` fires, stop the current sound immediately (call a `stopCurrentSound()` stub that terminates any in-flight oscillators) before starting the new reaction.
    - _Requirements: 2.2, 2.6_

  - [x] 2.2 Verify floating text duration is within [800, 1200] ms
    - Confirm `spawnFloatText` computes `850 + Math.random() * 300` for the `--dur` var. The closed interval is [850, 1150] ⊂ [800, 1200] — acceptable. Add an inline comment referencing requirement 2.3 and property 2 for traceability.
    - _Requirements: 2.3, 5.2 — Property 2_

  - [x] 2.3 Verify consecutive-duplicate re-roll logic
    - Confirm the `do/while` loop in `selectReaction` re-rolls once when `idx === lastReactionIndex` and stops after 3 tries. Add an inline comment referencing requirement 2.5 and property 3.
    - _Requirements: 2.5 — Property 3_

  - [x] 2.4 Verify floating text DOM cleanup
    - Confirm every `spawnFloatText` call attaches an `animationend` listener that calls `el.remove()`, and that this listener fires correctly for both normal completion and animation override. Add an inline comment referencing requirement 5.3 and property 11.
    - _Requirements: 5.3 — Property 11_

- [~] 3. Checkpoint — verify the patched `index.html` manually
  - Open `index.html` in a browser. Pet the cat at least 10 times to trigger a milestone. Verify pool validation error by temporarily shortening the `REACTIONS` array in dev tools. Verify reduced-motion by enabling the OS setting. Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Set up test harness
  - [x] 4.1 Initialise `package.json` and install test dependencies
    - Run `npm init -y` in the workspace root. Install `vitest`, `fast-check`, `@playwright/test`, and `jsdom` as dev dependencies with exact versions. Add `"test": "vitest --run"` and `"test:e2e": "playwright test"` scripts.
    - _Requirements: (testing infrastructure)_

  - [x] 4.2 Create `src/logic.js` — extracted pure functions
    - Extract the following pure/testable functions from `index.html` into `src/logic.js` as named exports (do not modify `index.html` — duplicate the logic):
      - `selectReaction(purrCount, lastReactionIndex, reactions, milestoneReaction)`
      - `calcFloatDuration(randomValue)` — `850 + randomValue * 300`
      - `initPurrCount(rawStorageValue)` — returns 0 for invalid/negative input
      - `validateReactionPool(reactions)` — returns `{ valid: boolean, error?: string }`
      - `selectMessage(messages, randomValue)` — picks a message by index
    - _Requirements: 2.1, 2.3, 2.5, 3.5, 4.3, 5.5_

  - [~] 4.3 Create `src/logic.test.js` — unit tests for extracted logic
    - Write example-based unit tests for:
      - `selectReaction` returns milestone reaction when `purrCount % 10 === 0`
      - `selectReaction` returns a pool member when `purrCount % 10 !== 0`
      - `initPurrCount` returns 0 for `null`, `'abc'`, `'-1'`, `'3.5'`
      - `initPurrCount` returns the integer for `'42'`
      - `validateReactionPool` returns invalid for arrays shorter than 8
      - `validateReactionPool` returns valid for a pool of 10 with all required categories
      - `calcFloatDuration` stays within [800, 1200] for inputs 0 and 1
    - _Requirements: 2.1, 3.5, 4.3, 5.5_

- [ ] 5. Property-based tests
  - [~] 5.1 Write property test for reaction selection always returns a pool member
    - **Property 1: Reaction selection always returns a pool member**
    - **Validates: Requirements 2.1**
    - Use `fc.array(reactionArb, {minLength:1})` and `fc.integer({min:1}).filter(n => n % 10 !== 0)` as generators; assert the returned reaction is contained in the pool.
    - Tag comment: `// Feature: pet-the-cat, Property 1`
    - _File: `src/logic.test.js`_

  - [ ]* 5.2 Write property test for floating text duration range
    - **Property 2: Floating text duration stays within [800, 1200] ms**
    - **Validates: Requirements 2.3, 5.2**
    - Use `fc.double({min:0, max:1})` to stub the random value; assert `calcFloatDuration(r)` is in `[800, 1200]`.
    - Tag comment: `// Feature: pet-the-cat, Property 2`
    - _File: `src/logic.test.js`_

  - [ ]* 5.3 Write property test for re-roll avoiding consecutive duplicate
    - **Property 3: Re-roll avoids immediate consecutive duplicate**
    - **Validates: Requirements 2.5**
    - Use `fc.array(reactionArb, {minLength:2})` and `fc.integer({min:0})` for lastIndex; assert result index differs from lastIndex (or pool has only 1 item).
    - Tag comment: `// Feature: pet-the-cat, Property 3`
    - _File: `src/logic.test.js`_

  - [~] 5.4 Write property test for pool size and category coverage
    - **Property 4: Reaction pool meets minimum size and category coverage**
    - **Validates: Requirements 3.1, 3.2**
    - Static assertion against the `REACTIONS` constant imported from `src/logic.js`; check `length >= 8`, unique names, and all five categories present.
    - Tag comment: `// Feature: pet-the-cat, Property 4`
    - _File: `src/logic.test.js`_

  - [~] 5.5 Write property test for milestone override on every multiple of 10
    - **Property 5: Milestone override fires on every multiple of 10**
    - **Validates: Requirements 3.3**
    - Use `fc.integer({min:1, max:100000}).filter(n => n % 10 === 0)`; assert `selectReaction` returns `MILESTONE_REACTION`.
    - Tag comment: `// Feature: pet-the-cat, Property 5`
    - _File: `src/logic.test.js`_

  - [ ]* 5.6 Write property test for no 3 consecutive repeated reactions
    - **Property 6: No reaction appears 3 or more consecutive times**
    - **Validates: Requirements 3.4**
    - Simulate a sequence of N pet events (N via `fc.integer({min:20, max:200})`), collecting reaction indices; assert no three consecutive indices are equal.
    - Tag comment: `// Feature: pet-the-cat, Property 6`
    - _File: `src/logic.test.js`_

  - [~] 5.7 Write property test for invalid pool blocking initialisation
    - **Property 7: Invalid pool blocks initialisation**
    - **Validates: Requirements 3.5**
    - Use `fc.array(reactionArb, {maxLength:7})`; assert `validateReactionPool` returns `valid: false`.
    - Tag comment: `// Feature: pet-the-cat, Property 7`
    - _File: `src/logic.test.js`_

  - [~] 5.8 Write property test for purr counter increments by exactly 1
    - **Property 8: Purr counter increments by exactly 1 per pet**
    - **Validates: Requirements 4.2**
    - Use `fc.integer({min:0, max:1e9})` for initial count; verify `count + 1` equals result.
    - Tag comment: `// Feature: pet-the-cat, Property 8`
    - _File: `src/logic.test.js`_

  - [~] 5.9 Write property test for purr counter init from localStorage
    - **Property 9: Purr counter initialises from localStorage**
    - **Validates: Requirements 4.3, 4.5**
    - Use `fc.oneof(fc.integer({min:0}).map(String), fc.string())` as generator; assert valid non-negative integers parse correctly, everything else returns 0.
    - Tag comment: `// Feature: pet-the-cat, Property 9`
    - _File: `src/logic.test.js`_

  - [ ]* 5.10 Write property test for message selection from pool
    - **Property 12: Message selection always returns a pool member**
    - **Validates: Requirements 5.5**
    - Use `fc.array(fc.string(), {minLength:3})` and `fc.double({min:0, max:1})` for the random stub; assert the result is a member of the messages array.
    - Tag comment: `// Feature: pet-the-cat, Property 12`
    - _File: `src/logic.test.js`_

- [~] 6. Checkpoint — run unit and property tests
  - Run `npm test` and ensure all tests pass with zero failures. Ensure all tests pass, ask the user if questions arise.

- [ ] 7. DOM and integration tests
  - [~] 7.1 Write JSDOM unit tests for floating text DOM cleanup and aria attributes
    - Use `jsdom` + `vitest` to test:
      - **Property 11**: after `animationend` fires on a float-text element, the element is no longer in `document.body` — use `fc.string()` as message generator.
      - **Property 13**: `#cat-wrap` always carries `aria-label="Pet the cat"`, `role="button"`, `tabindex="0"` on the rendered HTML.
    - Tag comments: `// Feature: pet-the-cat, Property 11` and `// Feature: pet-the-cat, Property 13`
    - _File: `src/dom.test.js`_

  - [ ]* 7.2 Write property test for localStorage persistence (Property 10)
    - Mock `localStorage` in JSDOM; use `fc.integer({min:0})` for the initial count; after simulating `onPet`, assert `localStorage.getItem('purrCount')` equals the incremented count as a string.
    - **Property 10: Purr counter persists synchronously after each pet**
    - **Validates: Requirements 4.4**
    - Tag comment: `// Feature: pet-the-cat, Property 10`
    - _File: `src/dom.test.js`_

  - [~] 7.3 Set up Playwright config and write integration tests
    - Create `playwright.config.js` targeting `chromium`, `firefox`, `webkit`. Set `testDir: './e2e'`.
    - Create `e2e/cat.spec.js` with tests:
      - Cat is visible and centered at viewport widths 320, 375, 768, 1440, 2560px.
      - Cat bounding box ≥ 200×200 at viewport > 375px; ≥ 150×150 at 375px.
      - No horizontal scrollbar at any of the above viewports.
      - `aria-label`, `role`, `tabindex` attributes present on `#cat-wrap`.
      - Pet via keyboard (Tab to focus + Enter) produces a visible counter increment.
      - Counter increments and remains visible without scrolling after a pet.
      - App loads with no console errors.
    - _Requirements: 1.1, 1.2, 1.3, 6.1, 6.2, 6.3, 6.4_

  - [ ]* 7.4 Write integration test for touch event response time
    - In `e2e/cat.spec.js`, simulate a `touchstart` event via Playwright's `tap()` API and assert the counter increments within 50ms (using `page.evaluate` with `performance.now()` before and after).
    - _Requirements: 6.2_

- [~] 8. Final checkpoint — full test suite
  - Run `npm test` (vitest unit + PBT) and `npm run test:e2e` (Playwright). All tests must pass. Ensure all tests pass, ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- `src/logic.js` duplicates logic from `index.html` for testability — `index.html` remains the single source of truth for the browser
- Property tests use `fast-check` with minimum 100 runs each (vitest default + `fc.assert` options)
- Each property test is tagged with `// Feature: pet-the-cat, Property N` for traceability to the design document
- Checkpoints ensure incremental validation at each phase
- All accessibility improvements are additive CSS/attribute changes — no structural HTML changes needed

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2", "1.3", "1.4"] },
    { "id": 1, "tasks": ["2.1", "2.2", "2.3", "2.4"] },
    { "id": 2, "tasks": ["4.1"] },
    { "id": 3, "tasks": ["4.2"] },
    { "id": 4, "tasks": ["4.3", "7.1"] },
    { "id": 5, "tasks": ["5.1", "5.2", "5.3", "5.4", "5.5", "5.6", "5.7", "5.8", "5.9", "5.10", "7.2"] },
    { "id": 6, "tasks": ["7.3"] },
    { "id": 7, "tasks": ["7.4"] }
  ]
}
```
