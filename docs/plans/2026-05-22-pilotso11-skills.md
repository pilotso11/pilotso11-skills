# pilotso11-skills Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Stand up a public GitHub repo `pilotso11/pilotso11-skills` that hosts a single Claude Code plugin, with CI that validates the plugin manifest and every skill on every push/PR, and main branch protection that blocks merging without a passing CI run.

**Architecture:** Single Claude Code plugin layout — one `.claude-plugin/plugin.json` at the repo root, a `skills/` directory holding one subdirectory per skill, each with a `SKILL.md`. CI is a single GitHub Actions job running `claude plugin validate .` in a `node:lts-slim` container with the Claude CLI installed from npm.

**Tech Stack:** GitHub Actions, `@anthropic-ai/claude-code` (npm), `gh` CLI, plain Markdown + JSON.

---

## Working directory

All file paths below are relative to `/Users/sethosher/projects/opensource/skills` (which is the local working copy and will become the `pilotso11/pilotso11-skills` repo). That directory already exists and is empty except for the `docs/plans/` directory containing the design doc and this plan.

---

### Task 1: Initialize the git repo

**Files:**
- Modify: `/Users/sethosher/projects/opensource/skills/` (run `git init`)

**Step 1: Initialize and configure**

```bash
cd /Users/sethosher/projects/opensource/skills
git init -b main
git config user.email "seth@oshers.com"
```

**Step 2: Verify**

Run: `git status`
Expected: "On branch main", with `docs/plans/*.md` listed under untracked files.

**Step 3: Commit checkpoint — do NOT commit yet**

The first commit happens in Task 7 once all the scaffolding is in place. This keeps the initial commit a single coherent "scaffold" change rather than fragments.

---

### Task 2: Create `.gitignore`

**Files:**
- Create: `.gitignore`

**Step 1: Write file**

```
# OS
.DS_Store
Thumbs.db

# Editors
.vscode/
.idea/
*.swp
*.swo
*~

# Node (in case contributors run tooling locally)
node_modules/
npm-debug.log*

# Claude Code local state
.claude/
```

**Step 2: Verify**

Run: `cat .gitignore | head -5`
Expected: First lines show `# OS` and `.DS_Store`.

---

### Task 3: Create `LICENSE`

**Files:**
- Create: `LICENSE`

**Step 1: Write standard MIT license text**

```
MIT License

Copyright (c) 2026 pilotso11

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

### Task 4: Create the plugin manifest

**Files:**
- Create: `.claude-plugin/plugin.json`

**Step 1: Create directory and write file**

```bash
mkdir -p .claude-plugin
```

`.claude-plugin/plugin.json`:
```json
{
  "name": "pilotso11-skills",
  "version": "0.1.0",
  "description": "Personal Claude Code skills by pilotso11",
  "author": {
    "name": "pilotso11",
    "url": "https://github.com/pilotso11"
  },
  "homepage": "https://github.com/pilotso11/pilotso11-skills",
  "license": "MIT",
  "keywords": ["claude-code", "claude-skills", "skills"]
}
```

**Step 2: Verify JSON is well-formed**

Run: `python3 -m json.tool .claude-plugin/plugin.json`
Expected: pretty-printed JSON matching the input, exit 0.

**Step 3: Validate with Claude CLI (the same check CI will run)**

Run: `claude plugin validate .`
Expected: "✔ Validation passed" (possibly with warnings about missing skills directory — those are fine; we add the skill next).

---

### Task 5: Create the placeholder `hello` skill

**Files:**
- Create: `skills/hello/SKILL.md`

**Step 1: Create directory and write skill**

```bash
mkdir -p skills/hello
```

`skills/hello/SKILL.md`:
```markdown
---
name: hello
description: Use when the user greets you (says "hi", "hello", "hey", etc.) to confirm the pilotso11-skills plugin is installed and active.
---

# Hello

Respond with a brief, friendly greeting and explicitly mention that the
`pilotso11-skills` plugin is loaded and active. This serves as a smoke test
that the plugin is wired up correctly.

Example response:

> Hello! 👋 The `pilotso11-skills` plugin is loaded — you're good to go.
```

**Step 2: Validate**

Run: `claude plugin validate .`
Expected: "✔ Validation passed" with output mentioning the manifest AND `skills/hello/SKILL.md` (skill is walked and frontmatter-checked).

---

### Task 6: Create the CI workflow

**Files:**
- Create: `.github/workflows/ci.yml`

**Step 1: Create directory and write workflow**

```bash
mkdir -p .github/workflows
```

`.github/workflows/ci.yml`:
```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:

jobs:
  validate:
    name: validate
    runs-on: ubuntu-latest
    container: node:lts-slim
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install Claude Code CLI
        run: npm install -g @anthropic-ai/claude-code@2

      - name: Show Claude version
        run: claude --version

      - name: Validate plugin manifest and skills
        run: claude plugin validate .
```

**Step 2: Lint the YAML**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))"`
Expected: no output, exit 0.

**Note for the engineer:** The single `claude plugin validate .` step satisfies both user-stated CI requirements ("skills load without error" + "library catalog loads without error") — verified during design that the command walks both the manifest and every `skills/*/SKILL.md`.

---

### Task 7: Create the README

**Files:**
- Create: `README.md`

**Step 1: Write the README**

`README.md`:
````markdown
# pilotso11-skills

Personal collection of [Claude Code](https://www.anthropic.com/claude-code)
skills authored by [@pilotso11](https://github.com/pilotso11), packaged as a
single installable plugin.

## Installing in Claude Code

### One-line install (recommended)

From within Claude Code, run the slash command:

```
/plugin install pilotso11-skills@https://github.com/pilotso11/pilotso11-skills
```

Claude Code will fetch the latest `main`, register the plugin, and load every
skill under `skills/`.

### Local development install

If you want to hack on a skill locally, clone the repo and launch Claude with
`--plugin-dir` pointing at the checkout:

```bash
git clone https://github.com/pilotso11/pilotso11-skills.git
cd pilotso11-skills
claude --plugin-dir .
```

Claude will load the plugin for that session only, picking up local edits to
any `SKILL.md` without reinstalling.

## Available skills

| Skill | Trigger | What it does |
|---|---|---|
| `hello` | User greets Claude | Confirms the plugin is loaded and active (smoke test) |

## Publishing a new skill

1. **Create the skill directory:**
   ```bash
   mkdir -p skills/<skill-name>
   ```
2. **Write `skills/<skill-name>/SKILL.md`** with YAML frontmatter. `name` and
   `description` are required; the description is what Claude uses to decide
   when to invoke the skill, so be specific about triggering conditions.
   ```markdown
   ---
   name: <skill-name>
   description: Use when ... (describe the triggering situation precisely)
   ---

   # Skill body

   Instructions for Claude when this skill is invoked.
   ```
3. **Validate locally:**
   ```bash
   claude plugin validate .
   ```
   Resolve any errors before opening a PR. Warnings are tolerated but worth
   addressing.
4. **Add your skill to the table** in this README under *Available skills*.
5. **Open a pull request.** CI will run the same `claude plugin validate .`
   check; it must pass before merge.
6. **Merge.** Once on `main`, anyone who installed the plugin will pick up
   the new skill on their next `claude plugin update`.

## CI

Every push to `main` and every pull request runs `claude plugin validate .`
inside a `node:lts-slim` container with the official `@anthropic-ai/claude-code`
CLI installed. The job fails if `.claude-plugin/plugin.json` is missing or
malformed, or if any `skills/*/SKILL.md` has invalid structure.

Branch protection on `main` requires this check to pass before merge.

## License

[MIT](LICENSE)
````

**Step 2: Verify**

Run: `wc -l README.md`
Expected: roughly 60–80 lines.

---

### Task 8: Initial commit

**Files:** all the above.

**Step 1: Stage and commit explicitly named files (not `-A`)**

```bash
git add .gitignore LICENSE README.md \
        .claude-plugin/plugin.json \
        skills/hello/SKILL.md \
        .github/workflows/ci.yml \
        docs/plans/2026-05-22-pilotso11-skills-design.md \
        docs/plans/2026-05-22-pilotso11-skills.md
git commit -m "$(cat <<'EOF'
feat: scaffold pilotso11-skills plugin

Initial layout for a Claude Code plugin holding personal skills:
- .claude-plugin/plugin.json — manifest
- skills/hello/ — placeholder skill so CI has something to validate
- .github/workflows/ci.yml — runs `claude plugin validate .` on push/PR
- README with install + contribution guide
- MIT license

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

**Step 2: Verify**

Run: `git log --oneline`
Expected: one commit, message starts with "feat: scaffold pilotso11-skills plugin".

Run: `git status`
Expected: "nothing to commit, working tree clean".

---

### Task 9: Create the public GitHub repo and push

**Step 1: Create + push in one command**

```bash
gh repo create pilotso11/pilotso11-skills \
  --public \
  --description "Personal Claude Code skills by pilotso11" \
  --source=. \
  --remote=origin \
  --push
```

**Step 2: Verify**

Run: `gh repo view pilotso11/pilotso11-skills --json url,visibility,defaultBranch`
Expected: JSON shows `"visibility": "PUBLIC"`, `"defaultBranch": "main"`, valid URL.

Run: `git remote -v`
Expected: `origin` pointing at `https://github.com/pilotso11/pilotso11-skills.git`.

---

### Task 10: Wait for first CI run and confirm it passes

This step is sequential — branch protection in Task 11 needs the `validate`
check name to be visible to GitHub, which only happens after at least one run.

**Step 1: Watch the workflow**

```bash
gh run watch --exit-status
```

(If no run is in progress yet, run `gh run list --limit 1` first to confirm
one is queued; if not, wait 10s and try `gh run list` again.)

Expected: the `validate` job exits with status `success`.

**Step 2: If CI fails**

- Read the failure: `gh run view --log-failed`
- Most likely cause: typo in `plugin.json` or `SKILL.md` frontmatter
- Fix locally, `git commit --amend` (only safe because nothing depends on the
  commit yet) or add a follow-up commit, `git push`, watch again
- Do NOT proceed to Task 11 until CI is green

---

### Task 11: Apply main branch protection

**Step 1: Apply the protection ruleset via the GitHub API**

```bash
gh api --method PUT /repos/pilotso11/pilotso11-skills/branches/main/protection \
  --input - <<'JSON'
{
  "required_status_checks": {
    "strict": true,
    "contexts": ["validate"]
  },
  "enforce_admins": false,
  "required_pull_request_reviews": {
    "required_approving_review_count": 0,
    "dismiss_stale_reviews": false,
    "require_code_owner_reviews": false
  },
  "restrictions": null,
  "required_linear_history": true,
  "allow_force_pushes": false,
  "allow_deletions": false,
  "block_creations": false,
  "required_conversation_resolution": false
}
JSON
```

**Step 2: Verify**

Run:
```bash
gh api /repos/pilotso11/pilotso11-skills/branches/main/protection \
  --jq '{ checks: .required_status_checks.contexts,
          strict: .required_status_checks.strict,
          pr_required: (.required_pull_request_reviews != null),
          linear: .required_linear_history.enabled,
          no_force: (.allow_force_pushes.enabled == false),
          no_delete: (.allow_deletions.enabled == false),
          admin_override: (.enforce_admins.enabled == false) }'
```

Expected output (key fields):
```json
{
  "checks": ["validate"],
  "strict": true,
  "pr_required": true,
  "linear": true,
  "no_force": true,
  "no_delete": true,
  "admin_override": true
}
```

---

### Task 12: Final sanity check

**Step 1: Confirm the install URL actually works**

From any directory:
```bash
claude plugin validate https://github.com/pilotso11/pilotso11-skills
```

Wait — `validate` only takes a local path. Instead, simulate the install by
cloning to a temp directory and validating:

```bash
TMPDIR=$(mktemp -d)
git clone --depth 1 https://github.com/pilotso11/pilotso11-skills.git "$TMPDIR/p"
claude plugin validate "$TMPDIR/p"
rm -rf "$TMPDIR"
```

Expected: "✔ Validation passed" — proves a fresh clone is a valid Claude Code
plugin (so the README install instructions will work for users).

**Step 2: Print the deliverables**

Report back to the user with:
- Repo URL
- A note that CI is green
- A note that branch protection is in place
- The exact `/plugin install` command they can paste into Claude Code

---

## Notes for the implementer

- Do not skip Task 10 (waiting for CI) before applying branch protection. If
  you try to require a status check that has never run, GitHub accepts the
  setting but the picker will not be able to autocomplete the name in the UI,
  and there is a small risk of typos.
- `claude plugin validate` exits 0 on warnings-only. That is acceptable for
  this repo. If you want stricter CI later, post-process the output with
  `grep -q "warning" && exit 1` — but only after the first warning-free green
  build, so we don't gate the initial push on it.
- The `docs/plans/` files (design + this plan) are intentionally committed.
  They document the why. Keep them.
