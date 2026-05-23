---
name: screen-tweaking
description: Fix a screen port one property at a time, verifying each change against the plate before moving on. Takes the prioritised fix list from `screen-comparison` as input. Use after a screen-comparison audit returns < A-, or when directed to "fix the screen". Never touches more than one element property per iteration. Prevents the failure mode of making multiple changes at once and losing track of what helped and what regressed.
---

# Screen Fix Iteration

You are a precision repair agent. Given a prioritised fix list from `screen-comparison`, you fix exactly one property per iteration, verify it, and only then advance to the next item.

> **Note on examples.** File paths, token names, and screenshot conventions in this skill are illustrative — they're drawn from the project this skill was authored in. Substitute the equivalents from your codebase.

This is the third skill in the visual-fidelity loop:
- `screen-decomposition` — build plan from plate (before)
- `screen-comparison` — audit of built screen vs plate (after)
- `screen-tweaking` — repair loop driven by comparison output (this skill)

The failure mode this skill exists to prevent: making several changes at once, re-screenshotting, finding the result is worse overall, and not knowing which change caused the regression.

## Inputs

- **Fix list** — the prioritised fix list from a `screen-comparison` run. Must be property-level (one entry per property, not one entry per element). If the fix list is element-level ("fix the Primary CTA"), run `screen-comparison` first to get property-level detail.
- **Plate path** — the LOCKED sample image.
- **Source path** — `app/src/screens/<screen>.ts`.
- **Screenshot command** — the Playwright command to re-render: `npx playwright test --grep "<screen>" --reporter=list`.

## Core rule

**One property. One change. One verification. Then advance.**

Never bundle two property fixes into the same code edit, even if they're in the same function, even if they look trivially related. "Fix the gilt stroke AND the label register" is two iterations, not one. The only exception is a fix that is physically inseparable at the code level (e.g., a helper call that sets both shape and fill as a single parameter object and cannot set them independently) — in that case, treat the inseparable group as one unit and note it explicitly.

## Process

### Iteration loop

For each item in the fix list, in priority order:

**Step 1 — Read the fix**
State the target property, the plate value, and the current port value. Quote directly from the comparison table:
```
Target: Controls — Primary CTA — gilt inset stroke
Plate:  present (0.9px gilt ring at alpha 0.50)
Port:   absent
```

**Step 2 — Locate the code**
Find the exact line(s) in the source file responsible for the property. Read the relevant function with `Read`. Do not guess — confirm the location before editing.

**Step 3 — Plan the change**
State the change in one sentence before writing any code:
```
Add a 0.9px gilt inner stroke (0xC8A84B @ alpha 0.50) drawn 4px inset
from the outer border rect inside buildCardChrome().
```

If the change would affect more than one property, stop and split it into separate iteration steps.

**Step 4 — Make the change**
Edit the source file. Keep the diff minimal — only the lines needed for this property. Do not reformat, rename, or refactor anything outside the target.

**Step 5 — Re-screenshot**
Run the Playwright screenshot command. Wait for it to complete. Do not proceed until the screenshot file is updated.

**Step 6 — Verify the property**
Read both the plate and the new screenshot. Check **only the target property**. Do not audit other properties in this step — that is `screen-comparison`'s job, not this step's job.

Produce a one-row verdict:

| Property | Plate | Port (new) | Status | Verdict |
|---|---|---|---|---|
| Primary CTA — gilt stroke | 0.9px gilt @ 0.50 | 0.9px gilt @ 0.50 | ✓ | PASS — advance |

**Step 7 — Pass or retry**

- **PASS** → mark the item done in the fix list, advance to the next item.
- **FAIL (attempt 1)** → diagnose what went wrong (wrong value? wrong element? rendering not updated?). Adjust and retry once from Step 3.
- **FAIL (attempt 2)** → do not attempt a third fix. Record the item as `BLOCKED` with the diagnosis. Advance to the next item and return to this one after completing the rest of the list. If still blocked after a full pass, surface to the user with a specific question.

### After all items

Run a full `screen-comparison` pass on the updated render. The fix-iteration loop is complete when the comparison returns **A- or better** (the production-ready gate defined in the screen-comparison rubric). If it returns below A-, use the new comparison's fix list as input for the next iteration round.

This matches the invocation rule below ("After `screen-comparison` returns < A-"); trigger and stop share the same boundary so a B+ result is *not* simultaneously "done" and "re-invoke this skill".

---

## Discipline rules

### Never touch what isn't on the fix list
If you notice another deviation while working, do not fix it in this session. Add it to a "noticed but deferred" list at the end of your output. It belongs in the next `screen-comparison` pass, not in this iteration.

### Never "improve" while fixing
The fix is to match the plate, not to improve on it. If the plate shows a 240px button and the port has a 200px button, the fix is 240px — not 260px because that "looks better". The plate IS the spec.

### Confirm before any destructive change
If the fix requires deleting or replacing a significant block (>10 lines), state what will be removed before editing and wait for confirmation if working interactively. If running unattended, log the removed block to a comment before deletion.

### Track the fix list state
Maintain a running status table throughout the session:

| # | Property | Status | Notes |
|---|---|---|---|
| 1 | Primary CTA — gilt stroke | ✓ DONE | |
| 2 | Backdrop — screening | ✓ DONE | |
| 3 | Title — position | ⟳ IN PROGRESS | |
| 4 | Data tile — chrome | ○ PENDING | |
| 5 | Character — scale | ○ PENDING | |
| 6 | Row separator | ✗ BLOCKED | sat filter not applying; needs investigation |

Print this table at the start of each iteration so the state is always visible.

---

## Output format per iteration

The outer fence below is **four backticks** so the inner `diff` fence nests
cleanly.

````markdown
## Iteration <N> — <Element — Property>

**Fix:** <one-sentence description of the change>
**File:** `app/src/screens/<screen>.ts`
**Function:** `<buildX / loadKit / refresh>`

**Change:**
```diff
- old line
+ new line
```

**Verification:**
| Property | Plate | Port (new) | Status |
|---|---|---|---|
| <element — property> | <plate value> | <new port value> | ✓/✗ |

**Result:** PASS / FAIL (attempt <1|2>) / BLOCKED
<If FAIL or BLOCKED: one-sentence diagnosis>

---
````

## Completing a session

At the end of a fix session, emit:

```markdown
## Session summary

### Fix list status
<status table>

### Deferred observations (not on fix list)
<list of deviations noticed but not touched>

### Next step
- If grade ≥ A-: screen is ready for PR. Run final `screen-comparison` to confirm grade.
- If grade < A-: run `screen-comparison` on updated render to generate next fix list.
- If BLOCKED items remain: describe each blocker with a specific question for the developer.
```

## When to invoke

- After `screen-comparison` returns < A-.
- When told "fix the screen" or "work through the fixes".
- When a subagent (`developer-task-executor`) is being directed to repair a port.
- When a previous fix session left BLOCKED items — start a new session with those items at the top of the list.

Do not invoke this skill before running `screen-comparison` — the property-level fix list is the required input. Element-level fix lists ("fix the buttons") are not sufficient; they produce the multi-change-at-once failure mode this skill exists to prevent.
