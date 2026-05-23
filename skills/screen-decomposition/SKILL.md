---
name: screen-decomposition
description: Read a locked sample plate and produce an implementation spec — a buildable decomposition of every visual element into its constructor call, layout coords, typography role, state binding, and asset source. Pair with `screen-comparison` (decomposition is the BEFORE; comparison is the AFTER). Use before opening a screen port PR, before briefing a subagent on a new screen, or when "I need to build this plate" is the task. Produces a markdown spec the developer (or `developer-task-executor`) builds against directly.
---

# Screen Decomposition

You are an implementation-plan author. Given a locked sample plate, you produce the **buildable spec** — the brief a developer (or subagent) can construct the screen against without going back to the image to make decisions.

> **Note on examples.** This skill is dense with concrete code patterns — Pixi.js calls, token names (`palette.ox`, `slots.S0`), helper functions (`drawFleur`, `makeButton`), asset filenames (`nbframe_color.svg`, `slim_frame.png`), MCP names (`vinsim-image`), and a `window.__vinsim.store` Playwright hook — drawn from the project this skill was authored in. The *structure* of the decomposition (frame strategy → regions → backdrop → card chrome → typography → controls → state) is general; substitute your stack's equivalents where examples show specifics.

This is the **inverse** of `screen-comparison`:
- `screen-comparison` audits a *built* screen against the plate (after).
- `screen-decomposition` produces the *build plan* from the plate (before).

A good decomposition closes the gap that creates the failures `screen-comparison` exists to catch: ambiguous "card on linen" without per-card chrome spec, "buttons" without variants, "backdrop" without screening strategy, "title" without font role.

## Inputs

- **Plate path** — the LOCKED sample image, e.g. `docs/redesign/art-kit/plates/samples/<screen>__LOCKED.{jpg,png}`
- **Screen id** — the new screen's id matching the `ScreenId` enum, e.g. `"warehouse"`.
- (Optional) **Existing port path** — if revising a screen, the current `app/src/screens/<screen>.ts` for diff context.

## Process

1. **Read the plate carefully.** Open with `Read`. Note every visible element — chrome, copy, typography, character, ornament, button — before writing.
2. **Extract the spatial skeleton.** Before any categorisation, do a dedicated spatial-only pass. See *Spatial skeleton* below. This is mandatory and populates the Layout Regions table with measured numbers, not slot labels.
3. **Lock the frame strategy.** Decide: is `nbframe_color.svg` screen-wide on this plate, or scoped to the central card? This drives every other decision (where titles can sit, how the backdrop interacts).
4. **Map regions to slots.** Assign S0/S1/S2 or CARD constants as *labels* for the skeleton regions already measured in step 2. Slot names are organisational; the measured % values are the spec.
5. **Inventory every atomic element** down the eleven categories below. Each entry includes: what it is, where it goes, which helper builds it, what state it reads.
6. **Decide asset sources.** For each non-trivial visual: reuse-from-kit / generate-via-MCP / draw-with-Graphics. Reuse first; generate only when a recognisable hand-drawn scene is required.
7. **Sequence the build.** Static layer constructed first (everything that doesn't react to state), dynamic layer second (state-driven), kit assets async-loaded last, frame on top.
8. **Specify the smoke test.** What state needs to be set before screenshot, and what should the snapshot show.
9. **Emit the markdown spec** using the template below.

---

## Spatial skeleton

This step exists because the taxonomy walk is category-driven and naturally reads screens semantically. Spatially implicit structure — column proportions, gutter widths, baseline grids, panel anchoring — is systematically skipped unless it is forced as its own pass. Do this pass first, in isolation, before categorising anything.

Slot names (`S0`, `S1`, `CARD`) are organisational labels only. They do not carry dimensional information. A developer handed `S0` without a width % has to guess. **Always record both.**

### How to build a skeleton

Describe the screen as a series of **regions and proportions only**. No semantic labels during this pass — name regions by position and role, not by content. Use approximate percentages of screen width/height.

Required measurements:
- **Major regions** — list each major panel/column/rail with `x offset %`, `width %`, `y offset %`, `height %`
- **Gutter widths** — estimated % of screen width between each pair of adjacent columns
- **Header / footer bands** — height as % of screen height
- **Hero element** — its bounding box as % of its containing region
- **Vertical rhythm** — approximate equal-spacing unit between repeated rows or items (in px or % if estimable)
- **Alignment axis** — is there a clear left-edge or centre-axis that most elements lock to?

Example skeleton output (abbreviated):

```
PLATE SKELETON
- Left column: x=0%, w=34%, full height
- Right column: x=36%, w=64%, full height
- Gutter: ~2% between columns
- Header band: y=0%, h=11%
- Hero card: within right column, y=14%, h=72%
- Footer strip: y=90%, h=10%
- Vertical rhythm (list rows): ~9% height each, ~1.5% gap
- Primary alignment axis: left edge at x=5% (both columns)
```

This skeleton feeds directly into the Layout Regions table in the output template. Every region row must carry both its slot label and its measured % values. A row with a slot name but no measurements is incomplete.

---

### Frame strategy
- **Screen-wide ornate** (Bench / Conception / Hub / Harvest / Cellar / WineBible / Warehouse / History): load `/kit/nbframe_color.svg` with `data:{resolution:3}` and `addChild` last.
- **Card-only ornate** (Grape Market / Char Select): no full-screen frame. The card itself gets a beveled outer + inset rectangle + small corner fleurs.
- **No frame**: rare; only if the plate's edge is meant to bleed into the surrounding chrome of the next screen.

Record the choice up-front; it constrains where titles can sit (clear of frame ornaments) and whether backdrops bleed to edges.

## Taxonomy — eleven decision categories

### 1. Frame strategy
(Already locked above — record choice + rationale here.)

### 2. Region / layout grid
- For 3-column screens: use `slots.S0` / `slots.S1` / `slots.S2` as labels. Record each column's measured `x %`, `w %`, and `y`-range from the skeleton.
- For single-card screens: define `CARD_X / CARD_Y / CARD_W / CARD_H / CARD_R` constants matching the plate's measured proportions, not estimated values.
- For sub-grids (lot columns inside a card, critic-card stacks): define gap + pad constants derived from skeleton rhythm measurement.
- For full-bleed atmospheric screens (Char Select, Grape Market): backdrop fills `screen.w × screen.h`, dialog floats above — record dialog bbox as % of screen.

**Rule:** every constant in this section must be derivable from the skeleton measurements. If a value cannot be grounded in a skeleton measurement, flag it as an estimate and note which element to re-measure.

### 3. Backdrop strategy
For each screen:
- **None** — plain `palette.linen` ground (Bench, Cellar instrument register).
- **Programmatic Graphics** — sky gradient + hills + buildings drawn in `Graphics` primitives. Use only when the plate's atmosphere is minimal or stylised geometric.
- **Generated SVG** — call `vinsim-image` MCP with a C-2 style prompt anchored on `2324b94d-b6b0-4218-9c1d-ddae93742152:GENERATED`. Matte + vectorise with `filter_speckle=12 + color_precision=3` (more aggressive for backgrounds where micro-detail reads as noise). Sanitise. Load with `data:{resolution:2}` and add as the first child of `staticLayer`.
- **Screening pass** (always with generated backdrops over a dialog): apply `ColorMatrixFilter` (saturation × 0.7) + an `alpha 0.15-0.2` linen overlay rect between the backdrop and the dialog so the dialog dominates.

Specify the choice + the prompt (if generated) + the screening pass.

### 4. Card chrome (per dialog)
For every floating card / dialog / sub-card, specify:
- Outer body: `roundRect` fill (linen / linen2 / inst).
- Drop shadow (optional): `+7,+9` offset, `0x000000 @ 0.30`.
- Outer border: `2-2.8 px` stroke, `palette.ox`.
- Inner inset: `4-8 px` margin, `0.9-1.4 px` gilt stroke at `0.40-0.60` alpha.
- Corner fleurs: 4 × small `drawFleur` glyphs at the card corners if the plate shows them.
- Header band (for stall-style sub-cards): top 36 px filled `palette.ox`, white-cream text.
- Inset data plaque (for prices/scores): 60-70 px tall sub-rect with `BADGE_BG`-style dark fill, top stripe coloured to denote grade.

### 5. Typography roles
For every text element, specify the helper + size + fill + extras:
- **Title** — script italic plate → `titleText(s, { size: 36-44, fill: palette.ox })`. Block-caps plate → `uiText(s, { size: 26-36, bold: true, track: 4, fill: palette.ox })`.
- **Section caption** — block-caps tracked → `uiText(s, { size: 11-12, fill: palette.ox, track: 3-4 })`.
- **Body** — `uiText(s, { size: 11-13, fill: palette.ink })`.
- **Italic display** ("Left Bank / Bordeaux") → `lblText(s, { size: 19-22, fill: palette.ox, italic: true })`.
- **Speech / Vinny voice** — `spkText(s, { size: 12-14, italic: true, fill: palette.ink })`.
- **Numerals** — `numericText(s, { size: 18-44, bold: true, fill: palette.gold | palette.ox })`. Anchor `"end"` for right-aligned scores.
- Anchors: `"middle"` for centred; `"end"` for right-aligned. Default left.
- **Title clearance**: anchor titles at `slots.S0.y0 + 14` minimum on screen-wide-frame screens to clear the gilt grape-corner ornament.

### 6. Controls (use shared Button when available)
Decide each control's variant:
- **Primary CTA**: `Button.primary` — oxblood pill, gilt inner stroke, gpale caps text. Size: 200-280 × 44-54.
- **Secondary CTA**: `Button.secondary` — outlined ox stroke, ox text, no fill. Size: ~120 × 32-40.
- **Wax-seal commit**: `Button.waxSeal` — circular oxblood medallion with gilt double-rule, "CONFIRM" caps inside, optional fleur left.
- **Slider track + handle**: 2 px ink stroke + 6 r gilt-on-ink dot.
- **Chip** (for state cycling): dark inset with gilt outline, light caps text.
- **Drag handle on a sprite**: container `eventMode = "static"` + cursor `"ns-resize"` or `"ew-resize"`.

Until the shared Button lands (DS-1), spec the geometry inline but note "TODO: migrate to Button.<variant>" so the cross-screen sweep is trackable.

### 7. Information presentation
- **Lot/stall row** chrome: header band + region sub-line + thin gilt rule + data plaque.
- **Stat plaque** (REPUTATION/COFFERS/etc.): caption above + icon medallion (wax seal / coin purse) + value lblText + tier word in caps below.
- **Speech bubble**: `roundRect linen2 + gilt stroke` + small triangle pointing at speaker + `spkText` italic body inside.
- **Critic card** (Reveal): `roundRect linen2` + `circle` portrait + name caption + score numeral + italic pull-quote.
- **Ledger page** (Conception / History): cream parchment rect + spine line + handwritten-style entries with ruled lines.

### 8. Iconography
- **Vine leaf** — kit `/kit/leaf{1..4}.svg` (white-fill, Pixi-tintable to lot colour).
- **Grape cluster** — drawGrapeCluster helper (6 dots in triangle + stem + leaf).
- **Wax seal** — drawWaxBadge helper (circle + double-rule + grape-laurel imprint).
- **Coin purse** — drawCoinPurse helper (ellipse + neck + knot).
- **Risk shield** — drawRiskShield helper (pentagonal shield + crossbar tick).
- **Fleur-de-lis** — drawFleur helper (almond + ball).
- **Diamond ornament** — 4-point diamond at gilt rule midpoint.

For each icon: where it sits, size, tint, and whether to draw or import.

### 9. Characters (figure assets)
Pose registry — map game-state to pose SVG:
- **Bench** — `alma.svg` (= alma_03b-compose-right, journal + pen, right-facing). Crop via `texture.frame` to bbox (0.32, 0.07, 0.50, 0.88).
- **Conception** — `mentor_elder.svg` (right-facing wine glass), placed inside the central window plate.
- **Hub** — `alma.svg` + `mentor_elder.svg`, two figures conversing in foreground.
- **Cellar** — `alma_02-decide.svg` (at desk) + `mentor_elder.svg` portrait inset.
- **Reveal** — verdict-mapped pose: VINDICATION → `alma_04`, MIXED → `alma.svg`, QUIET_REGRET → `alma_05`.
- **Wine Bible** — `alma_07-study.svg`.
- **Label Studio** — `alma_08-bottling.svg`.
- **Warehouse** — `alma_08-bottling.svg` (foreground winemaker).
- **Char Select** — protagonist carousel: `proto_f_young / proto_m_young / proto_f_elder / proto_m_elder`.
- **History** — no figure (the ledger is the protagonist).

For each: load via `Assets.load<Texture>`, crop via `texture.frame = new Rectangle(...)` to the measured figure bbox, size by `targetH` (preserving ratio), position. Never mirror via `scale.x = -1` (uncanny — wine glass in the wrong hand). If the plate wants the other direction, regenerate the pose.

### 10. State bindings
For each dynamic element, declare:
- Read: which `AppState` field drives the value (`vintage.lots[i].pct`, `estate.cashTier`, `vintage.thesis.targetId`, etc.).
- Write: which `store` setter the interaction calls (`store.setLotPct`, `store.setHarvestStance`, `store.navigate`, `store.confirmConception`, etc.).
- Guards: which `*State` machine value should gate the input (`vintage.benchState === "AIMING"`, `vintage.conceptionState !== "COMMITTED"`).
- Refresh: which fields the `subscribe` callback refreshes (per-row pct text, balance numeral, etc.).

### 11. Accessibility & smoke test
- Every interactive Container: set `eventMode`, `cursor`, `accessible = true`, `accessibleTitle`, `accessibleHint`, `tabIndex`.
- Add a Playwright test that navigates to the screen, waits 2500 ms, screenshots to `test-results/<NN>-<screen>.png`, and asserts `getState().currentScreen === "<screen>"`.

## Output template

```markdown
# <Screen> — implementation spec (decomposed from <plate path>)

## 1. Frame strategy
<screen-wide ornate | card-only ornate | none>
**Rationale:** <one sentence>

## 2. Spatial skeleton
<region table with % measurements — populated before any slot labels are assigned>

| Region | x % | w % | y % | h % | Notes |
|---|---|---|---|---|---|
| Left column | 0% | 34% | 0% | 100% | |
| Right column | 36% | 64% | 0% | 100% | |
| Gutter | 34% | 2% | — | — | between columns |
| Header band | 0% | 100% | 0% | 11% | |
| … | | | | | |

Vertical rhythm unit: <~9% row height, ~1.5% gap>
Primary alignment axis: <left edge at x=5%>

## 3. Layout regions (slot labels + measured values)
| Region | Slot label | x % | w % | y % | h % | Contents |
|---|---|---|---|---|---|---|
| Left column | S0 | 0% | 34% | 4% | 92% | figure, stat plaques |
| Right column | S1 | 36% | 64% | 4% | 92% | card, buttons |
| … | | | | | | |

*Every constant below derives from the skeleton above. Estimates are flagged.*

## 4. Backdrop
- Type: <none | programmatic Graphics | generated SVG>
- (If generated) Prompt: `…`
- Screening: <none | ColorMatrix sat 0.7 + linen overlay alpha 0.18>

## 5. Card chrome
For each card:
- Outer rect: …
- Drop shadow: …
- Outer border: …
- Inner inset: …
- Corner fleurs: <yes/no>
- Header band: …

## 6. Asset manifest
| Asset | Source | Notes |
|---|---|---|
| `/kit/nbframe_color.svg` | reuse | data.resolution=3 if used screen-wide |
| `/kit/<bg>.svg` | generate | $0.04 cost; prompt above |
| `/kit/poses/<alma_xx>.svg` | reuse | crop bbox … |
| `/kit/mentor_elder.svg` | reuse | … |
| (others) | … | … |

## 7. Element table — every text + control + icon + tile

| # | Element | Helper | Geometry | State binding | Notes |
|---|---|---|---|---|---|
| 1 | Title "<COPY>" | `uiText / titleText` (size, fill, anchor, track) | (x, y) | static / `<store field>` | … |
| 2 | Caption "<COPY>" | … | … | … | … |
| 3 | Lot row (×4) | composite | rowL, ry, … | `vintage.lots[i].pct` ↔ `setLotPct` | … |
| … | | | | | |

Group by static (built once in `buildStatic`) vs dynamic (built in `buildDynamic` and updated in `refresh`).

## 8. Controls
| Control | Variant | Position | Action | Guard |
|---|---|---|---|---|
| Primary CTA | Button.primary | … | `store.<setter>` then `navigate("<screen>")` | `<machine>State === "<value>"` |
| Secondary | Button.secondary | … | … | … |
| Slider × N | track+handle | … | `store.setLotPct(id, t*100)` | `benchState === "AIMING"` |

## 9. Build sequence
1. `bg` rect — `palette.linen`
2. `staticLayer` add to root
3. `dynamicLayer` add to root
4. `buildStatic()` — title, captions, card chrome, region headers, rules
5. `buildDynamic()` — text placeholders, rows, plaques, buttons (state-bound)
6. `void loadKit()` — async: frame + figures + generated backdrops
7. `unsubscribe = store.subscribe(refresh)`
8. `refresh()` — initial state apply

## 10. State bindings (summary)
- Reads: …
- Writes: …
- Guards: …

## 11. Accessibility
| Element | accessibleTitle | accessibleHint | tabIndex |
|---|---|---|---|
| … | … | … | … |

## 12. Smoke test
```ts
test("<Screen>: renders <key element>", async ({ page }) => {
  await page.setViewportSize({ width: 1344, height: 768 });
  await page.goto("/");
  await waitForHook(page);
  await page.evaluate(() => {
    window.__vinsim.store.<setup>();
    window.__vinsim.store.navigate("<screen>");
  });
  await page.waitForTimeout(2500);
  expect(await page.evaluate(() => window.__vinsim.store.getState().currentScreen)).toBe("<screen>");
  await page.screenshot({ path: "test-results/<NN>-<screen>.png" });
});
```

## 13. File deliverables
- New: `app/src/screens/<screen>.ts`
- Edit: `app/src/screens/router.ts` (add import + case)
- Edit: `app/tests/e2e/smoke.spec.ts` (add the test above)
- (If generating) New: `docs/redesign/art-kit/plates/backdrops/<screen>_bg/*` + `app/public/kit/<screen>_bg.svg`
- (If new pose) New: `docs/redesign/art-kit/poses/*` + `app/public/kit/poses/*.svg`

## 14. Definition of done
- `npx tsc --noEmit` clean
- `npx playwright test --reporter=list` — all green
- `screen-comparison <screen>` returns **B+ or better** (use this for self-check before claiming done)
- Committed with the project's commit-message style
- Pushed via `gh auth git-credential` to `phase-c-build`
```

## Author principles

- **Reuse over generate.** Check `app/public/kit/` and `app/public/kit/poses/` first. Generate (cost $0.04 / image) only when the plate's scene cannot be assembled from existing pieces and matters for atmosphere (Grape Market village, Char Select dusk vineyard, future Hub estate).
- **Spec the chrome, not just the elements.** "A card" is not a spec — "linen card with 2 px ox border + 0.9 px gilt inset at alpha 0.48 + drop shadow +7,+9 at alpha 0.30 + corner fleurs" is a spec.
- **Slot names are labels, not dimensions.** `S0`, `S1`, `CARD` are organisational identifiers. They carry no width or height information on their own. Every constant in the layout spec must be traceable back to a measured % from the spatial skeleton. If a value can't be grounded in a measurement, flag it explicitly as an estimate.
- **Per-screen frame choice is load-bearing.** Decide screen-wide vs card-only FIRST. Half of the screen-comparison "majors" come from getting this wrong.
- **Screen-back generated backdrops.** Without the sat × 0.7 + linen-overlay pass, the dialog drowns.
- **Build order matters.** Static first, dynamic second, kit async last, frame on top. Mixing the order causes z-fights and "the frame ate my title".
- **Game-state-driven imagery.** Reveal swaps the Alma pose by verdict. Conception's mentor changes by agency rung (apprenticeship → mentorship). Bake this into the pose registry, not into each screen separately.

## When to invoke

- Before writing any `app/src/screens/<new-screen>.ts`.
- When briefing a subagent (`developer-task-executor`) for a port — the decomposition becomes the brief verbatim.
- When revising a screen — produce a fresh decomposition, then diff against the existing source to find the gap.
- When a screen-comparison audit returns < B+ and the fix list is non-trivial — re-decompose from scratch.

The output of this skill is the *brief* and the *acceptance criteria*. Pair with `screen-comparison` post-build: decomposition forward, comparison backward. Together they close the visual-fidelity loop without leaving room for impressionistic shortcuts.
