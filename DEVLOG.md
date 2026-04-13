# 13 Flames — Development Log

## Project Overview

**13 Flames** is a ritual web app based on a winter solstice intention-setting practice.
The user writes 13 personal intentions on virtual pieces of paper, folds and seals them,
then burns one each night from the Winter Solstice (Dec 21) through New Year's Eve.
On the 13th night (Jan 2), the final paper is opened to reveal a task to carry into the new year.

Single file app: `index.html` — self-contained HTML/CSS/JS, no external dependencies.
State is persisted in `localStorage` under the key `13flames_v2`.

---

## Session Log — April 2026

### Feature 1 — Initial Build
**Prompt:** Read the document I wrote and create a website that I can turn into an app.

**Source doc (`13 flames Plan.docx`) summary:**
- 13 pieces of paper; user types intention on each and "folds" it with animation
- Once all 13 sheets recorded, an emblem seals each one
- From Winter Solstice sunset → Jan 1: select a paper, light a virtual candle/fire, burn it
- Last paper: seal broken, paper opened, task revealed
- Suggested task database if user doesn't know where to start
- Fire options: campfire, candle (with color)
- Background options: dusk, night with stars, table, wood floors

**Built:**
- Single `index.html` with embedded CSS and JS
- 13 paper cards on a dark starry background with parchment texture
- Click-to-open writing modal (lined notebook look with red margin line)
- "Fold & Tuck Away" button saves intention to localStorage
- Animated wax seal SVG (16-pointed starburst with flame motif) pops onto each card once all 13 are written
- Campfire and candle fire scenes with CSS flame animations
- Floating ember particles on campfire
- Burn animation (paper darkens bottom-up, fire edge rises, paper disappears)
- Reveal phase: last paper opens with a 3D unfold animation
- 30-item suggested task database (positive, achievable tasks)
- Settings panel: fire type, candle color (10 swatches), background (4 options)
- Animated star canvas for night background
- Phase preview buttons for testing (Writing / Burning / Reveal)

---

### Feature 2 — Envelope Strip + Drag to Flame
**Prompt:** Don't reveal the intention before releasing to the flame. Instead of a button, allow dragging the folded paper to the flame to burn. On the burn screen make the hidden papers small envelopes with a wax seal.

**Changes:**
- Burning phase strip: replaced flat sealed cards with CSS-drawn envelopes
  - Triangular sealed flap at top (dark brown `#8a5c20`)
  - X-crease fold lines on body (diagonal gradients simulating back of sealed envelope)
  - Wax seal SVG centered on front
  - Roman numeral label in bottom-right corner
- Replaced burn button with drag-and-drop:
  - Desktop: `dragstart` / `dragover` / `drop` events
  - Mobile/touch: `touchstart` / `touchmove` / `touchend` with ghost clone element
  - Fire drop zone glows with dashed ring when hovering
- When envelope is dropped on fire:
  - Small envelope clone appears over fire and plays `consumeEnv` animation (rises, shrinks, fades)
  - Fire flares up briefly (`scaleX(1.4) scaleY(1.5)`)
  - Paper marked as burned, strip re-renders
- Intention text is never shown in the burning phase — kept sealed inside envelope

---

### Feature 3 — Date Rules + Day Counter
**Prompt:** Writing page should be openable until the solstice. After which all envelopes are sealed. You should not be able to burn more than 1 per day unless you have missed a day. There should be a counter in the top right showing days until reveal. That should be the 13th day.

**Date logic:**
- Solstice = Dec 21 (sunset = 6 pm) → writing closes
- Burning period = Dec 21 sunset → Jan 1
- Reveal = Day 13 = Jan 2 (Dec 21 + 12 days)
- `getBaseYear()` handles year rollover (January returns previous year's solstice)
- `dayNumber()` → 1-indexed (1 = Dec 21, 13 = Jan 2), clamped 1–13
- `papersAllowedToBurn()` = `min(dayNumber, 12)` — catches up missed days naturally
- `canBurnMore()` = `papersAllowedToBurn() > papersBurned()`
- `burnedDate` (ISO date string) stored on each paper when burned

**UI changes:**
- Top-right cluster: gold day counter stacked above gear button
- Counter shows during burning phase only; hides on other phases
- Counter displays `daysUntilReveal()` (0–12) with "day/days until your reveal"
- Burn instruction text updates dynamically:
  - Normal: "Drag a sealed envelope to the flame to release it."
  - Catch-up: "You may release N papers today — drag one to the flame."
  - Locked: "You've released your paper for today — return tomorrow." (text turns orange-red)
- Fire drop zone dims (`opacity: .55`, `pointer-events: none`) when locked
- Envelopes get `.no-burn` class (cursor: not-allowed) when locked
- All papers auto-sealed when entering burning phase (blank papers sealed as empty intentions)
- Writing grid shows "The solstice has passed. Your papers are sealed." after solstice
- `isDemoMode = true` flag bypasses all date restrictions for the preview buttons

---

### Feature 4 — Jan 1 Early Reveal Option
**Prompt:** Allow opening on Jan 1 after a reminder that the intention is to open tomorrow on the 13th night.

**Logic:**
- `canEarlyReveal()` = `daysUntilReveal() === 1 && papersBurned() >= 12`
  (Jan 1 AND all 12 burns complete; `isDemoMode` also enables it)
- When true, the last remaining envelope gets `.last-paper` class:
  - Pulsing gold glow animation (`lastGlow` keyframes)
  - "Open tonight" label appears beneath it
  - Cursor changes to pointer
- Clicking it opens the early reveal confirmation modal:
  - 🕯️ icon, heading "One more night"
  - Message: "The intention of this ritual is to open your final paper on the **13th night — January 2nd**."
  - Two buttons: "Wait until tomorrow" (dismisses) | "Open tonight" (triggers reveal phase)
- Jan 2+: app auto-advances to reveal phase as before (no prompt)

---

### Feature 5 — Paper Folding Animation
**Prompt:** Can we use an animation to fold the intentions that have been written.

**Implementation:**
- Full-screen dark overlay (`#foldOverlay`, `z-index: 450`) appears when "Fold & Tuck Away" is clicked
- Writing modal closes first; fold animation plays over the grid
- Overlay shows a 320×420px paper sheet with:
  - Top half: first portion of the user's written text (italic, lined paper texture, red margin line)
  - Bottom half: remaining text; this is the half that folds
  - Crease shadow line at center
- Animation sequence:
  1. **0–220ms**: paper is shown flat (user sees their words)
  2. **220–970ms**: bottom half rotates `rotateX(-180deg)` around its top edge (3D perspective fold) — underside of paper (darker cream `#e2cfa0`) visible mid-rotation
  3. **1100–1550ms**: entire folded paper gets `.shrink-away` class → scales to 0.1, shoots upward, fades out
  4. **1600ms**: overlay hidden, grid re-renders with card in folded state
- Uses `transform-style: preserve-3d`, `perspective: 1600px`, `backface-visibility: hidden` on both faces
- Seal pop animation plays after all 13 are folded

---

## Technical Architecture

```
index.html
├── <style>          — all CSS embedded (~350 lines)
│   ├── backgrounds (night/dusk/table/floor)
│   ├── stars canvas
│   ├── header + phase badge + top-right cluster
│   ├── writing phase + paper cards
│   ├── fold animation overlay
│   ├── seal pop animation
│   ├── burning phase (fire drop zone, campfire, candle)
│   ├── envelope styles
│   ├── reveal phase
│   └── modals (writing, settings, early reveal, fold overlay)
│
├── <body>           — HTML structure
│   ├── #starsCanvas
│   ├── .app (bg class toggled by settings)
│   │   ├── header (title, subtitle, phase badge, tr-cluster)
│   │   ├── #writingPhase (papers grid)
│   │   ├── #burningPhase (fire drop zone + envelope strip)
│   │   └── #revealPhase (reveal paper + task)
│   ├── #foldOverlay
│   ├── #writingModal
│   ├── #settingsModal
│   └── #earlyRevealModal
│
└── <script>         — all JS embedded (~600 lines)
    ├── Data (ROMAN, TASKS[30], CANDLE_COLORS[10])
    ├── State (localStorage key: 13flames_v2)
    ├── Date utilities (getBaseYear, dayNumber, daysUntilReveal, canBurnMore, etc.)
    ├── Stars canvas (requestAnimationFrame loop)
    ├── Seal SVG generator
    ├── Fire renderers (campfireHTML, candleHTML)
    ├── Render functions (renderGrid, renderStrip, renderReveal, renderFire)
    ├── Drag & drop (desktop + touch, fire drop zone)
    ├── playFoldAnimation()
    ├── triggerBurn()
    ├── switchPhase()
    ├── Modal handlers (writing, settings, earlyReveal)
    └── init()
```

## State Shape

```json
{
  "papers": [
    { "id": 0, "text": "", "folded": false, "sealed": false, "burned": false, "burnedDate": null }
    // × 13
  ],
  "settings": {
    "fireType": "campfire",
    "candleColor": "#f0e68c",
    "background": "night"
  },
  "phase": "writing",
  "revealTaskIndex": null
}
```

## Ritual Calendar Reference

| Date      | Day # | Event                                      |
|-----------|-------|--------------------------------------------|
| Dec 21    | 1     | Solstice sunset — writing closes, burning begins |
| Dec 22    | 2     | Burn paper 2                               |
| …         | …     | …                                          |
| Dec 31    | 11    | Burn paper 11                              |
| Jan 1     | 12    | Burn paper 12; early reveal available (with warning) |
| Jan 2     | 13    | **Reveal** — 13th night, final paper opens |

## Local Dev Server

```bash
cd /Users/ErinDaniels/Desktop/13flames
python3 -m http.server 8080
# open http://localhost:8080
```
