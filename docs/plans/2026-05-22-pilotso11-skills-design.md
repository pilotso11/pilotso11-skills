# pilotso11-skills — Design

**Date:** 2026-05-22
**Status:** Approved
**Repo:** `pilotso11/pilotso11-skills` (public, MIT)

## Goal

A public GitHub repo to hold Claude Code skills authored by `pilotso11`, structured
as a single Claude Code plugin so users can install it once and pick up every
skill it ships. CI must validate that the plugin manifest and every skill load
without error, so that broken contributions cannot reach `main`.

## Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Layout | Single plugin, many skills | User selected during brainstorm; simpler than a marketplace and easy to grow |
| CI image | `node:lts-slim` + `npm i -g @anthropic-ai/claude-code` | Anthropic does not publish an official `claude-code` container; npm is the official distribution |
| Validation command | `claude plugin validate .` | Confirmed it validates both `plugin.json` and every `skills/*/SKILL.md` and exits non-zero on real errors |
| License | MIT | Matches reference `designer-skills` repo and most public `pilotso11` repos |
| Branch protection | PR required, CI required, linear history, no force-pushes, admin override allowed | User selected |

## Repository layout

```
pilotso11-skills/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── hello/
│       └── SKILL.md
├── .github/
│   └── workflows/
│       └── ci.yml
├── docs/
│   └── plans/
│       └── 2026-05-22-pilotso11-skills-design.md  (this file)
├── README.md
├── LICENSE
└── .gitignore
```

## File contracts

### `.claude-plugin/plugin.json`
Plugin manifest. Required keys: `name`, `version`, `description`. Optional but
included for hygiene: `author`, `homepage`, `license`, `keywords`.

### `skills/hello/SKILL.md`
Placeholder skill so CI has a real artifact to validate from the first commit.
Must include YAML frontmatter with `name` and `description`. Body describes
what Claude should do when the skill triggers.

### `.github/workflows/ci.yml`
- Triggers: `push` to `main`, `pull_request` against any branch
- Single job `validate`, runs in `node:lts-slim`
- Steps: checkout → install pinned Claude CLI → `claude --version` →
  `claude plugin validate .`
- Job name is exposed as a required status check on `main`

### `README.md`
Sections:
1. What this is
2. Installing in Claude Code (plugin install + local --plugin-dir flows)
3. Available skills
4. Publishing a new skill (step-by-step contributor guide)
5. CI summary
6. License

### `LICENSE`
Standard MIT text with `Copyright (c) 2026 pilotso11`.

### `.gitignore`
Node-style ignores (`node_modules/`, `.DS_Store`, editor swap files) — we do
not run JS here but contributors may have local tools.

## CI behavior

`claude plugin validate .` produces:
- **Exit 0** if no errors, even if there are warnings
- **Exit 1** if `plugin.json` is missing/malformed or a `SKILL.md` has hard
  errors

This single command satisfies both checks the user asked for:
- "skills load without error" — every `skills/*/SKILL.md` is walked and
  frontmatter-validated
- "library catalog loads without error" — `plugin.json` is schema-validated

Tested locally against good and broken plugins on Claude Code `2.1.142`.

## Branch protection rules

Applied via `gh api PUT /repos/pilotso11/pilotso11-skills/branches/main/protection`:

```json
{
  "required_status_checks": { "strict": true, "contexts": ["validate"] },
  "enforce_admins": false,
  "required_pull_request_reviews": {
    "required_approving_review_count": 0,
    "require_code_owner_reviews": false
  },
  "required_linear_history": true,
  "allow_force_pushes": false,
  "allow_deletions": false,
  "restrictions": null
}
```

`enforce_admins: false` so the owner can self-merge in emergencies.
`required_approving_review_count: 0` so the owner can open and merge their own
PRs (CI is still gating).

## Implementation order

1. Scaffold all files locally
2. `git init` + initial commit
3. `gh repo create pilotso11/pilotso11-skills --public --source=. --push`
4. Wait for first CI run to complete (so the `validate` check name exists in
   GitHub's required-status-check picker)
5. Apply branch protection via `gh api`
6. Verify protection settings

## Open questions / future work

- If skill count grows past ~20, revisit migrating to a marketplace structure
  with topical sub-plugins.
- Could add a `scripts/new-skill.sh` helper to scaffold a `SKILL.md`; deferred
  until there's a second skill in the repo.
- Could add markdownlint or a frontmatter linter as a second CI step; deferred
  until `claude plugin validate` proves insufficient in practice.
