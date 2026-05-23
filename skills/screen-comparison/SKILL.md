---
name: screen-comparison
description: Granular side-by-side audit of a rendered screen vs its locked sample plate. Inventories every visual element (frame, layout, typography, controls, info presentation, iconography, characters, atmosphere, palette, game-system consistency), marks plate vs port match per element, and produces a categorised score + a prioritised fix list. Use when reviewing a screen port, when asked "how does this compare to the sample", or before declaring a screen "done".
---

# Screen Comparison

You are a UI critique specialist. The goal is **granular, structured audit** of a rendered screen against its locked sample plate. Generic "feels off" or impressionistic feedback is the failure mode this skill exists to prevent.

> **Note on examples.** Specific file paths (e.g. `app/src/screens/...`, `/kit/...`), token names (`palette.ox`, `slots.S0`), and asset filenames throughout this skill are illustrative — drawn from the project this skill was authored in. Substitute the equivalents from your codebase.

## Inputs

- **Plate path** — the LOCKED sample image, e.g. `docs/redesign/art-kit/plates/samples/<screen>__LOCKED.{jpg,png}`
- **Render path** — the latest Playwright screenshot, e.g. `app/test-results/<NN>-<screen>.png`
- **Source path** (optional) — the TS file backing the screen, e.g. `app/src/screens/<screen>.ts`. Read for control wiring detail when needed; the audit itself reads from the rendered pixels.

## Process

1. **Open both images** with the `Read` tool. Study the plate first, then the render. Do not write any output until both have been read.
2. **Build the spatial skeleton** — BEFORE the taxonomy walk, do a dedicated spatial-only pass on the plate. See *Spatial skeleton* below. This is mandatory and cannot be skipped.
3. **Diff the skeletons** — build the same spatial skeleton for the render, then produce an explicit side-by-side diff. Layout mismatches must surface here as numbers, not impressions.
4. **Inventory the plate** — walk every category in the taxonomy below and note what's in the plate (presence, style, size, count, position). Don't yet look at the render while doing this — the goal is to anchor the comparison in *what the design demands*, not in *what was built*.
5. **Inventory the render** against the same taxonomy.
6. **Produce the comparison table** — one row **per property** of each element, not one row per element. See *Property decomposition* below. Columns: Element — Property / Plate / Port / Status / Notes. Status uses ✓ match, ~ partial, ✗ miss, + extra-not-in-plate.
7. **Score by category** — pass / minor / major / missing. Sum to an overall grade.
8. **Prioritised fix list** — one entry per failing property, ordered by visible-impact ÷ effort. Top item first. Each entry must name the exact property and the exact change needed.

---

## Spatial skeleton

This step exists because the taxonomy walk is category-driven and naturally reads screens semantically. Spatially implicit structure — column proportions, gutter widths, baseline grids, panel anchoring — is systematically skipped unless it is forced as its own pass. Do this pass first, in isolation, before touching any other category.

### How to build a skeleton

Describe the screen as a series of **regions and proportions only**. No semantic labels allowed in this pass — name regions by position and role, not by content. Use approximate percentages of screen width/height.

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

### Skeleton diff

After building both skeletons, produce a table:

| Region | Plate | Port | Delta | Layout status |
|---|---|---|---|---|
| Left column width | 34% | 25% | −9pp | ✗ |
| Right column width | 64% | 73% | +9pp | ✗ |
| Header height | 11% | 11% | 0 | ✓ |
| Hero card top | 14% | 18% | +4pp | ~ |
| Row rhythm | ~9% | ~12% | +3pp | ~ |

**Layout grading rules** (applied to skeleton diff, independent of category scores):
- Any region delta ≥10 percentage points → automatic **major** on Composition, regardless of how it looks at a glance.
- Any region delta 5–9 pp → **minor** on Composition unless every other category passes, in which case it stays minor.
- Delta <5 pp → acceptable; does not penalise Composition on its own.
- A missing region (present in plate, absent in port) → automatic **missing** on Composition.
- Vertical rhythm off by >25% of the plate rhythm unit → **minor**; off by >50% → **major**.

These rules are **hard overrides**. If the skeleton diff shows a ≥10pp delta, the Composition score cannot be `pass` no matter how good the rest of the screen looks. The skeleton numbers take precedence over visual impression.

### Grading discipline on layout specifically

Layout proportion deviations are **not a matter of taste**. Do not soften a layout finding because:
- The screen "looks balanced" despite the deviation — the plate IS the balance spec.
- The delta seems small in absolute pixels — use the percentage rules above.
- The content fills the space "well enough" — fill quality is irrelevant to proportion fidelity.

If you find yourself writing "~ partial" on a layout region, ask: *is the delta actually under 5pp?* If not, it is `✗`.

---

## Property decomposition

This is the key discipline that prevents coarse rows from hiding real failures.

**The rule:** `~ partial` is not a permitted status on a multi-property element. If you're tempted to write `~ partial` for "Primary CTA", stop — break the CTA into its constituent properties and mark each one individually. Partial match on what, exactly?

Every complex element must be decomposed into atomic property rows. The format is `Element — property`. Examples:

```
| Controls | Primary CTA — shape        | pill      | pill         | ✓ |                    |
| Controls | Primary CTA — fill         | oxblood   | oxblood      | ✓ |                    |
| Controls | Primary CTA — gilt stroke  | present   | absent       | ✗ | missing inset ring |
| Controls | Primary CTA — width        | ~240 px   | ~200 px      | ~ | 17% too narrow     |
| Controls | Primary CTA — label caps   | ALL CAPS  | Title Case   | ✗ | wrong register     |
| Controls | Primary CTA — position     | x=58%     | x=58%        | ✓ |                    |
```

### Property sets per element type

Use these as the minimum property set. Add rows for any additional properties visible in the plate.

**Button (primary or secondary):** shape, fill, border/stroke, label register, label text, size (w×h), position (x%, y%)

**Card / dialog:** outer fill, outer border stroke+colour, drop shadow, inner inset stroke, corner ornaments, header band

**Typography element:** font role (script/block/body), size, colour, case, tracking, anchor (left/centre/right), position (x%, y%), visible/clipped

**Character figure:** which pose, facing direction, scale (% of column height), crop tightness, x-position in column

**Data tile / plaque:** background fill, top stripe colour, border, value text size+colour, label text size+colour, icon present/absent

**Backdrop:** type (SVG/Graphics/none), saturation (screened back / full), linen overlay present/absent, coverage (full-bleed / card-only)

**Row item (in a list):** height, separator style, left icon, label, value, badge/plaque chrome

**Stat plaque:** caption present, icon type+size, value numeral size+colour, tier word present, border

### When `~ partial` IS acceptable

Only on properties where the deviation is genuinely continuous and hard to classify as pass or fail — e.g. a colour that's close but slightly warm, a size that's within 5–10%. In that case, the Notes column must quantify: *"~15% wider than plate"* not *"slightly off"*.

---

### Grading discipline (read carefully — failure mode this skill exists to prevent)

A `+ extra not in plate` row is **NOT free**. Adding chrome the plate
doesn't show — even "on brand" chrome like a wax seal, a separator
rule, a season chip, or an extra primary CTA — is a deviation from the
design contract. Score these the same way you'd score a `✗ miss`:

- One `+ extra` in a category = `minor`.
- Two or more `+ extra` in a category, OR a `+ extra` that changes
  the visual centre of gravity (e.g., adding a primary button where
  the plate has a caption + arrow) = `major`.

The plate IS the spec. If chrome wasn't in the plate, justify removing
it or downgrade the screen. The rubric never lets "extra polish" mask
deviations.

A "structural match" alone never exceeds **C**. To reach **B** you
need atmosphere + frame + palette correct AND ≤2 categories at minor.
To reach **A-** you need ≤1 minor and zero majors AND zero
load-bearing `+ extra` rows that change composition or visual
hierarchy. **A** requires every category at pass with zero `+ extra`.

## Taxonomy — eleven categories, every element accounted for

### 1. Composition
- Layout regions (columns, panels, gutters) — what proportion of the screen does each region take?
- Visual hierarchy / eye-flow — what reads first, second, third?
- Negative space — is the screen breathable or crowded?
- Centring vs offset — is the hero element centred or anchored to a side?

*Note: the spatial skeleton diff feeds directly into the Composition score. Apply the layout grading rules above before writing the Composition row.*

### 2. Frame & chrome
- **Screen-level frame** — is there a screen-wide border? Style (ornate gilt / plain / none)?
- **Card-level frames** — does each dialog/card have its own border? Style?
- **Ornaments** — corner pieces, fleur-de-lis, ribbons, medallions, seals — list each and where.
- **Rules and dividers** — gilt lines, double-rule borders, inset rectangles inside cards.
- **Common slip:** using `nbframe_color.svg` at full screen size when the plate uses it only around the central card. Check this every time.

### 3. Typography
- **Title** — font family (script italic vs sans block-caps), size, case, tracking.
- **Heading hierarchy** — h1/h2/h3 distinct levels.
- **Body text** — font, size, line-height adequacy.
- **Labels and captions** — small-caps tracked? regular? italic?
- **Numerals** — display weight, alignment (centered, baseline-aligned with units).
- **Hand-written / signature** — is there a script italic register for "voice" copy?
- **Title clipping** — frequent slip: titles anchored at `slots.S0.y0` get covered by the gilt frame's ornaments. Verify visible position.

### 4. Controls
- **Primary button** — shape (pill / rounded-rect / wax-seal medallion), fill (oxblood / gold), label register, presence of inset gilt stroke.
- **Secondary buttons** — outlined vs filled, smaller-scale.
- **Disabled state** — is one shown? colour treatment?
- **Slider / drag handle** — track style, handle shape, dot diameter.
- **Toggle / radio / checkbox** — presence and style.
- **Text input** — outline, fill, placeholder treatment.
- **Tabs / chips** — pill chips for cycling state.
- **Cross-screen consistency** — does the same button style match the Bench's CTA? Flag deviations explicitly.

### 5. Information presentation
- **Data tiles** — quality grades, prices, scores — inset chrome (wooden plaque effect) or plain text on linen?
- **Lists / rows** — bullet style (gilt dot / wax circle / sketched check), row separators (gilt thin line or none).
- **Tables** — column headers, row banding.
- **Stat plaques** — Reputation badge, Cash purse — fully chromed (border + icon + value + tier word)?
- **Pull quotes / speech bubbles** — bubble shape (rounded rect / cloud), tail direction, italic body.

### 6. Iconography
- **Icon register** — line, sketched, filled, gilt-on-oxblood?
- **Custom illustrations** — grape clusters, vine leaves, wax seals, scrolls, barrels. Each as a separate inventory line.
- **Icon alignment** — baseline-aligned with adjacent text? Sized proportionally to the label?

### 7. Characters
- **Which protagonist** — Alma v1 / mentor_elder / one of proto_*. Does the chosen pose match the plate's character?
- **Pose direction** — left-facing / right-facing / three-quarter. A mirror via `scale.x = -1` is uncanny (wine glass in the wrong hand) — flag it.
- **Scale** — figure occupies what fraction of its column? Plate vs port.
- **Cropping** — is the texture frame tight on the figure, or is there padding?

### 8. Atmosphere
- **Backdrop type** — generated SVG / programmatic Graphics / plain linen / none.
- **Backdrop screening** — saturated full image vs desaturated+lightened+alpha-overlaid as context layer. The dialog should always dominate over the backdrop.
- **Lighting direction** — where is the warm light coming from? Pendant lanterns / window / no light?
- **Depth cues** — perspective lines, multiple silhouette layers, foreground-midground-background separation.
- **Mood register** — moody-cellar / sunlit-workshop / dusk-market / candlelit-archive / clinical-instrument. Does the screen feel like the plate's mood?

### 9. Palette
- **Primary surface** — warm parchment, cool linen, dark slate, walnut, dusk-blue.
- **Accents** — oxblood, gilt, gold, verdigris, walnut. Match the plate's accent set.
- **Text on surface** — ink-on-linen, gpale-on-instrument, gilt-on-oxblood.
- **Per-tile colour** — variety tints on flasks/leaves, critic-card sealing, lot-row badges.
- **Temperature** — overall: does the screen *feel* warm or cool? Plate vs port.

### 10. State & interaction
- **Which states render** — default / selected / hover / disabled / committed.
- **Affordances** — cursor changes, drag indicators, focus rings.
- **Accessibility chrome** — `accessibleTitle` / `accessibleHint` / `tabIndex` set on interactive elements (verify in source).
- **Empty-state copy** — when data is absent ("Cycle to choose"), does it read as a hint or as broken UI?

### 11. Game-system consistency
- **Button family** — primary/secondary across screens use the same shape + chrome + label style?
- **Title family** — across screens, titles share the same register (script-italic OR block-caps), not mixed?
- **Frame use** — every screen treats the gilt frame the same way (screen-wide vs card-only is a *design-decision per screen*, but the frame asset itself should be uniform).
- **Iconography** — same wax-seal shape, same leaf glyph, same coin badge wherever they appear?
- **Spacing rhythm** — slot system from `tokens.ts` honoured (S0/S1/S2)?

## Output template

```markdown
# <Screen> — comparison vs <plate file>

## Plate spatial skeleton
<region table with % measurements>

## Render spatial skeleton
<region table with % measurements>

## Skeleton diff
<delta table with layout status per region>

## Plate inventory (what the design demands)
<short paragraph: composition, mood, dominant elements>

## Render inventory (what was built)
<short paragraph>

## Per-property table

| Category | Element — Property | Plate | Port | Status | Notes |
|---|---|---|---|---|---|
| Composition | Left column — width | 34% | 34% | ✓ | |
| Composition | Right column — width | 64% | 64% | ✓ | |
| Composition | Hero card — top offset | 14% | 18% | ~ | +4pp drift |
| Frame & chrome | Screen frame — scope | card-only | screen-wide | ✗ | wrong frame strategy |
| Frame & chrome | Card border — outer stroke | 2px ox | 1px ox | ~ | too thin |
| Frame & chrome | Card border — inner inset | present | absent | ✗ | |
| Frame & chrome | Corner ornaments | 4× fleur | absent | ✗ | |
| Typography | Title — font role | script italic | block caps | ✗ | wrong helper |
| Typography | Title — size | 40px | 40px | ✓ | |
| Typography | Title — position | y=8% | y=4% | ✗ | clipped by frame |
| Controls | Primary CTA — shape | pill | pill | ✓ | |
| Controls | Primary CTA — fill | oxblood | oxblood | ✓ | |
| Controls | Primary CTA — gilt stroke | present | absent | ✗ | |
| Controls | Primary CTA — width | ~240px | ~200px | ~ | ~17% narrow |
| Controls | Primary CTA — label register | ALL CAPS | Title Case | ✗ | |
| Controls | Slider — track style | 2px ink | 2px ink | ✓ | |
| Controls | Slider — handle | 6r gilt dot | 4r gold dot | ~ | wrong colour |
| Info | Data tile — background | dark plaque | plain text | ✗ | no chrome |
| Info | Data tile — top stripe | grade-coloured | absent | ✗ | |
| Info | Row separator | gilt thin rule | none | ✗ | |
| Info | Stat plaque — icon | wax seal | absent | ✗ | |
| Icons | Grape cluster | present | absent | ✗ | |
| Characters | Protagonist — pose | alma_03b | alma_03b | ✓ | |
| Characters | Protagonist — facing | right | right | ✓ | |
| Characters | Protagonist — scale | 78% col h | 55% col h | ✗ | too small |
| Atmosphere | Backdrop — type | generated SVG | programmatic | ✗ | wrong asset type |
| Atmosphere | Backdrop — screening | screened | unscreened | ✗ | dialog drowned |
| Palette | Surface — temperature | warm parchment | cool linen | ✗ | missing warm tint |
| Palette | Accents — gilt | present | absent | ✗ | |
| State | Default state | renders | renders | ✓ | |
| State | Empty-state copy | hint copy | absent | ✗ | |
| Consistency | Primary CTA — matches Bench | yes | no | ✗ | different shape |

## Category scores

| Category | Score | Rationale |
|---|---|---|
| Composition | pass / minor / major / missing | … |
| Frame & chrome | … | … |
| Typography | … | … |
| Controls | … | … |
| Information | … | … |
| Iconography | … | … |
| Characters | … | … |
| Atmosphere | … | … |
| Palette | … | … |
| State & interaction | … | … |
| Game-system consistency | … | … |

## Overall grade: <A | B+ | B | B- | C+ | C | C- | D | F>

## Prioritised fix list

Each entry targets one property. This list feeds directly into `screen-fix-iteration`.

1. **<Element — Property>** — exact change required. File: `app/src/screens/<screen>.ts`, function: `<buildX>`.
2. **<Next>** — …
3. **<Next>** — …
4. (Lower priority items.)
```

## Grading rubric

- **A** — Every property passes with zero `+ extra` deviations. The port could sit next to the plate and a reasonable observer wouldn't immediately reject it.
- **A-** — At most one minor; zero majors; zero load-bearing `+ extra` rows that change composition or visual hierarchy.
- **B** — Atmosphere + frame + palette all correct AND ≤2 categories at minor. No majors.
- **C** — Half the categories have major issues OR multiple `+ extra` rows that add chrome the plate doesn't show. Recognizable as the same screen, but the design's *intent* is undersold.
- **D** — Wireframe-level fidelity. Right boxes in roughly right places but no chrome, no atmosphere, no palette match.
- **F** — Broken state — missing elements, wrong layout, would mislead a user about the screen's purpose.

A "structural match" alone never exceeds **C**. The plate IS the spec.
Extra chrome — even "on-brand" extras — is a deviation, not a pass.

## Common slips to check every time

These are recurring patterns from the project this skill was authored in; treat
the specifics as examples and adapt to your codebase's equivalents.

- `nbframe_color.svg` used screen-wide where the plate scopes it to a card.
- Generated backdrop sprites not desaturated / not screened back, so they fight the dialog for attention.
- Titles anchored at `slots.S0.y0` get clipped by the gilt frame ornaments. Move down ~30 px or switch to a card-only-frame screen.
- `titleText` (script italic via Italianno) used where the plate shows block-caps sans. `uiText` with `bold + upper + track 4` is the right register.
- Per-card frames missing — cards are just `rect + linen2 + 1 px gilt stroke` instead of beveled outer + inset rect + corner notches.
- Plain text where the plate has inset chrome on data tiles (price plaques, quality grades, scores).
- Cold linen palette used for screens whose plate is warm parchment — add a warm-tan tint layer.
- Programmatic landscape graphics where the plate has a hand-drawn estate — generate an SVG backdrop via the MCP and use that.
- Buttons differ in shape, size, fill, and label register from screen to screen. Flag as game-system consistency issue.
- Sprite poses cropped to plate-of-figure size are then sized to a column smaller than the figure; figure renders as a tiny blob in the corner. Use `texture.frame` with measured bbox.

## When to invoke

- After porting a new screen, BEFORE declaring it done.
- When asked "how does this compare to the sample".
- When a port has been claimed complete but the user pushes back on visual fidelity.
- Before opening a revision PR — to ground the diff in specific elements.

The output of this skill is the *brief* for `screen-fix-iteration`. Each row in the prioritised fix list maps to one iteration of that skill.
